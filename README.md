package com.sarvatra.rtsp.service;

import com.sarvatra.rtsp.transaction.TokenTxnConstants;
import org.apache.commons.lang3.exception.ExceptionUtils;
import org.jpos.core.Configuration;
import org.jpos.core.ConfigurationException;
import org.jpos.iso.ISOUtil;
import org.jpos.q2.QBeanSupport;
import org.jpos.util.LogEvent;
import org.jpos.util.Logger;
import org.jpos.util.NameRegistrar;

import java.util.Date;
import java.util.concurrent.*;

public abstract class BaseThreadBasedPollingService extends QBeanSupport implements TokenTxnConstants, Runnable {

    public static final String LOG_SEPARATOR = "*******************************************************************************";

    boolean enabled;
    Long delayInMilliseconds;
    String transactionName;
    String institute;
    String transactionManagerQueue;
    int threadCount;
    String product;
    ScheduledThreadPoolExecutor threadPoolExecutor;


    @Override
    public void setConfiguration(Configuration cfg) throws ConfigurationException {
        super.setConfiguration(cfg);
        institute = cfg.get("institute", "1");
        threadCount = cfg.getInt("thread-count", 10);
        delayInMilliseconds = cfg.getLong("delay-in-milliseconds", 45000);
        transactionName = cfg.get("txn-name");
        enabled = cfg.getBoolean("enable", false);
        transactionManagerQueue = cfg.get("txn-manager-queue", "RTSP.SYS.TXN");
        product = cfg.get("product", "SYS");
    }

    @Override
    public void run() {
        while (running()) {
            try {
                getLog().info("Starting service : "+getName());
                process();
                getLog().info("Processing completed : "+getName());
            } catch (Exception e) {
                getLog().error("Exception in main thread " + ExceptionUtils.getStackTrace(e));
            }
            getLog().info("Now sleeping the "+getName()+" for "+delayInMilliseconds+" ms");
            ISOUtil.sleep(delayInMilliseconds);
        }
    }

    @Override
    protected final void startService() {
        if (!enabled) {
            getLog().info("Service disabled via setenv");
        } else {
            getLog().info("Service started at " + new Date());
            new Thread(this).start();
        }
    }

    public abstract void process() throws InterruptedException;

    @Override
    protected void initService() throws Exception {
        NameRegistrar.register(getName(), this);
        threadPoolExecutor= new ScheduledThreadPoolExecutor(threadCount);
        LogEvent logEvt = new LogEvent();
        logEvt.addMessage(LOG_SEPARATOR);
        logEvt.addMessage(getName() + " service configuration");
        logEvt.addMessage(LOG_SEPARATOR);
        logEvt.addMessage("Enabled            |   " + this.enabled);
        logEvt.addMessage("Thread count       |   " + this.threadCount);
        logEvt.addMessage("Delay              |   " + this.delayInMilliseconds);
        logEvt.addMessage("Transaction Name   |   " + this.transactionName);
        logEvt.addMessage("Product            |   " + this.product);
        logEvt.addMessage(LOG_SEPARATOR);
        Logger.log(logEvt);

    }
    @Override
    protected void destroyService() throws Exception {
        super.destroyService();
        if (threadPoolExecutor != null) {
            threadPoolExecutor.shutdown();
            threadPoolExecutor = null;
        }
    }

    protected ScheduledThreadPoolExecutor getThreadPoolExecutor() {
        if (threadPoolExecutor == null) {
            threadPoolExecutor = new ScheduledThreadPoolExecutor(threadCount);
        }
        return this.threadPoolExecutor;
    }
}



















package com.sarvatra.rtsp.service;

import com.sarvatra.rtsp.cache.InstituteCache;
import com.sarvatra.rtsp.ee.MerchantPayoutDetails;
import com.sarvatra.rtsp.server.ApiSourceSchemaService;
import com.sarvatra.rtsp.transaction.TokenTxnConstants;
import com.sarvatra.rtsp.util.CommonData;
import org.apache.commons.lang3.StringUtils;
import org.apache.commons.lang3.exception.ExceptionUtils;
import org.jpos.space.Space;
import org.jpos.space.SpaceFactory;
import org.jpos.transaction.Context;
import org.jpos.util.Log;
import org.jpos.util.LogEvent;
import org.jpos.util.Logger;
import org.json.JSONObject;

import java.util.concurrent.CountDownLatch;

public class MerchantPayoutFeature implements Runnable, TokenTxnConstants {
    private final MerchantPayoutDetails payoutDetails;
    private final InstituteCache instituteCache;
    private final String transactionName;
    private final String txnManagerQueue;
    private final CountDownLatch latch;
    private final Log log;

    public MerchantPayoutFeature(MerchantPayoutDetails payoutDetails, InstituteCache instituteCache, String transactionName, String txnManagerQueue, CountDownLatch latch, Log log) {
        this.payoutDetails = payoutDetails;
        this.instituteCache = instituteCache;
        this.transactionName = transactionName;
        this.txnManagerQueue = txnManagerQueue;
        this.latch = latch;
        this.log = log;
    }

    @Override
    public void run() {
        LogEvent evt = log.createLogEvent("Payout details :" + payoutDetails.getPayeeMobile());
        try {
            //  Prepare payload
            JSONObject payload = preparePayload();
            //  Generate transaction ID for payout
            String txnID = CommonData.getSystemTxnUniqueId(instituteCache.getInstitution(), 35);
            //  Soft lock payout record
            updatePayoutRecord(txnID);
            //  Queue context for payout transaction manager
            processMerchantPayout(txnID, payload);
            evt.addMessage("Transaction submitted to transaction manager");
            //  Decrement latch counter
            latch.countDown();
        } catch (Exception e) {
            evt.addMessage("Exception in payout feature " + ExceptionUtils.getStackTrace(e));
        }
        Logger.log(evt);

    }

    private JSONObject preparePayload() {
        //  Prepare payload to be queued in context
        JSONObject payload = new JSONObject();
        payload.put("mpd-id", payoutDetails.getId());
        payload.put("txn-type", payoutDetails.getTxnType());
        payload.put("tran-type", StringUtils.upperCase(payoutDetails.getTxnType()));
        payload.put("refund-type", StringUtils.upperCase(payoutDetails.getTxnType()));
        payload.put("amount", payoutDetails.getAmount());
        payload.put("payee-vpa", payoutDetails.getPayeeVPA());
        payload.put("payee-mobile", payoutDetails.getPayeeMobile());
        payload.put("payer-mobile", payoutDetails.getPayerMobile());
        payload.put("merchant-id", payoutDetails.getMid());
        payload.put("org-txn-id", payoutDetails.getRefundOrgTxnId());
        payload.put("remarks", payoutDetails.getRefundOrgTxnId());
        payload.put("refund-type", payoutDetails.getRefundType());
        payload.put("cred-type", "PREAPPROVED");
        payload.put("denomination", payoutDetails.getLoadDenomination());
        payload.put("acc-num", payoutDetails.getPayeeAccountNumber());
        return payload;
    }

    private void updatePayoutRecord(String txnID) throws Exception {
        //  Soft lock payout record
        org.jpos.ee.DB.execWithTransaction(db -> {
            payoutDetails.setStatus("L");
            payoutDetails.setPayoutStatus("I");
            payoutDetails.setSwitchTxnId(txnID);
            db.saveOrUpdate(payoutDetails);
            return payoutDetails;
        });
    }
    private void processMerchantPayout(String txnId, JSONObject payload) {
        Space<String, Context> space = SpaceFactory.getSpace();
        Context ctx = new org.jpos.transaction.Context();
        ctx.put(TXNNAME, transactionName);
        ctx.put(TARGET_POJO, payload);
        ctx.put(POJO, payload);
        ctx.put("TXN_TYPE", this.payoutDetails.getTxnType());
        ctx.put(RAW_REQUEST, payload);
        ctx.put(INSTITUTE_CACHE, this.instituteCache);
        ctx.put(API_SOURCE, ApiSourceSchemaService.get(ApiSourceSchemaService.API_SOURCE_SYS));
        ctx.put(TXN_ID, txnId);
        space.out(txnManagerQueue, ctx);
    }
}














package com.sarvatra.rtsp.service;

import com.sarvatra.common.util.SarvatraDateUtil;
import com.sarvatra.rtsp.cache.InstituteCache;
import com.sarvatra.rtsp.ee.MerchantPayoutDetails;
import com.sarvatra.rtsp.manager.MerchantPayoutManager;
import com.sarvatra.rtsp.server.InstituteCacheHelper;
import org.apache.commons.lang3.exception.ExceptionUtils;
import org.jpos.util.LogEvent;
import org.jpos.util.Logger;

import java.util.*;
import java.util.concurrent.CountDownLatch;

public class PayoutTxnProcessService extends BaseThreadBasedPollingService {

    @Override
    public String toString() {
        return super.toString() + "~" + getName();
    }

    public void process() throws InterruptedException {
        InstituteCache instituteCache = InstituteCacheHelper.getInstitute(institute);
        try {
            List<MerchantPayoutDetails> payoutTxnList = org.jpos.ee.DB.exec(db ->
                    new MerchantPayoutManager(db).getMerchantPayoutTransactionList(new SarvatraDateUtil().getStartOfDay()));

             logPayoutSummary(payoutTxnList);

            if (payoutTxnList.isEmpty()) {
                getLog().info("No payout transactions found");
                return;
            }

            // Queue payout transactions
            CountDownLatch latch = new CountDownLatch(payoutTxnList.size());
            getLog().info("Payout Txn Details Size :"+payoutTxnList.size());
            payoutTxnList.forEach(payoutDetails -> getThreadPoolExecutor().execute(new MerchantPayoutFeature(payoutDetails, instituteCache, transactionName, transactionManagerQueue, latch, getLog())));

            latch.await();
            getLog().info("Done with merchant payout service");

        } catch (Exception ex) {
            getLog().info("Exception occurred: " + ExceptionUtils.getStackTrace(ex));
        }
    }


    private void logPayoutSummary(List<MerchantPayoutDetails> payoutTxnList) {
        LogEvent evt = getLog().createLogEvent("Merchant Payout Summary");
        evt.addMessage(LOG_SEPARATOR);
        evt.addMessage("Start Date        | " + new SarvatraDateUtil().getStartOfDay());
        evt.addMessage("Payout count      | " + payoutTxnList.size());
        evt.addMessage("Transaction Name  | " + this.transactionName);
        evt.addMessage(LOG_SEPARATOR);
        Logger.log(evt);
    }
}





<merchant-payout-service class="com.sarvatra.rtsp.service.PayoutTxnProcessService" logger="Q2"
                         realm="merchant-payout-txn-poll"
                         configuration-factory="org.jpos.util.GenericConfigurationFactory">
    <property name="institute" value="env:MERCHANT_PAYOUT_INSTITUTE:1"/>
    <property name="enable" value="env:MERCHANT_PAYOUT_SERVICE_ENABLE:false"/>
    <property name="txn-manager-queue" value="env:MERCHANT_PAYOUT_SERVICE_TXN_QUEUE:RTSP.SYS.TXN"/>
    <property name="txn-name" value="env:MERCHANT_PAYOUT_TXN_NAME:MerchantPayout.POOL"/>
    <property name="delay-in-milliseconds" value="env:MERCHANT_PAYOUT_DELAY:60000"/>
    <property name="thread-count" value="env:MERCHANT_PAYOUT_THREAD_COUNT:5"/>
    <property name="product" value="env:MERCHANT_PAYOUT_PRODUCT:APP"/>
    <property name="trace" value="true"/>
</merchant-payout-service>




<?xml version='1.0'?>
<!DOCTYPE SYSTEM [
        <!ENTITY prepare_and_open   SYSTEM "cfg/prepare_and_open.inc">
        <!ENTITY normalize   SYSTEM "deploy/normalize.inc">
        <!ENTITY sms-notification   SYSTEM "cfg/sms-notification.inc">
        <!ENTITY queue-sms-notification   SYSTEM "cfg/queue-sms-notification.inc">
        <!ENTITY onus_bank_debit_account   SYSTEM "deploy/debit_request.inc">
        <!ENTITY onus_bank_credit_account   SYSTEM "deploy/credit_request.inc">
        <!ENTITY check-risk-lite   SYSTEM "cfg/check-risk-lite.inc">
        <!ENTITY debit_reversal_request   SYSTEM "deploy/debit_reversal_request.inc">
        <!ENTITY onus_merchant_bank_credit_account   SYSTEM "deploy/credit_merchant_request.inc">
        <!ENTITY onus_bank_fetch_account   SYSTEM "deploy/account_list_request.inc">
        <!ENTITY upi-validate-vpa   SYSTEM "cfg/upi-validate-vpa.inc">
        <!ENTITY KID "bdk.001">
        ]>
<!-- This File embedded in the rtsp-common in the  -->
<rtsp-config-txnmgr class="org.jpos.transaction.TransactionManager" logger="Q2"
                    configuration-factory="org.jpos.util.GenericConfigurationFactory">
    <property name="queue" value="RTSP.SYS.TXN"/>
    <property name="sessions" value="env:TXNMGR_CONFIG_SESSIONS:20"/>
    <property name="debug" value="true"/>
    <property name="call-selector-on-abort" value="true"/>

    &prepare_and_open;

    <participant class="org.jpos.transaction.participant.Switch" logger="Q2" realm="Switch">
        <property name="ManageMerchant" value="init check-merchant ensure-merchant-details close"/>
        <property name="CreateMerchant" value="init int-app-api-meta check-merchant create-merchant sms-notify close"/>
        <property name="UpdateMerchant" value="init int-app-api-meta check-merchant update-merchant close"/>
        <property name="MerchantStatement" value="init load-merchant-customer merchant-statement close"/>
        <property name="MerchantSettlementTxnStatement"
                  value="init load-merchant-customer merchant-settlement-statement close"/>
        <property name="GetSettlementStatementDetails" value="init get-settlement-details close"/>
        <property name="ExternalFetchCustomerWallet" value="init load-customer-wallet prepare-wallet-response close"/>
        <property name="MerchantBalance" value="init load-merchant-customer merchant-balance close"/>
        <property name="CustomerBalance" value="init load-merchant-customer merchant-balance close"/>
        <property name="CustomerInvitee" value="init customer-invitee close"/>
        <property name="GetMerchantDetails" value="init merchant-details close"/>
        <property name="CheckMerchantStatus" value="merchant-status close"/>
        <property name="KycBaseLimitConfig" value="kyc-limit-config close"/>
        <property name="MerchantCommonPay.LOAD" value="init int-app-api process-payout-request close"/>
        <property name="MerchantManualRedeem"
                  value="init int-app-api parse-manual-redeem-request find-account handle-redeem-debit update-final-status close"/>
        <property name="MerchantCommonPay.REFUND" value="init int-app-api process-payout-request close"/>
        <property name="MerchantCommonPay.PAYOUT" value="init int-app-api process-payout-request close"/>
        <property name="MerchantCommonPay.DBT" value="init int-app-api process-payout-request close"/>
        <property name="MerchantCommonPay.CASHBACK" value="init int-app-api process-payout-request close"/>
        <property name="MerchantFinder" value="init merchant-location-finder close"/>
        <property name="MerchantLogIntent" value="init int-app-api-meta merchant-log-intent close"/>
        <property name="MerchantPayout.POOL" value="init int-app-api process-payout-request update-final-status close"/>
        <property name="UpiPayout.POOL" value="init int-app-api process-upi-payout-request close"/>
        <property name="MerchantCashback.REWARD"
                  value="init int-app-api process-payout-request save-db update-reward close"/>
        <property name="ExternalLoad"
                  value="init int-app-api check-duplicate parse-load-request load-customer-wallet fetch-token-wallet check-rules parse-issue-token populate-intermediate-response sms-notify prepare-response-external-load close"/>
        <property name="ExternalCheckStatus" value="init load-customer-wallet find-merchant-txn close"/>
        <property name="RtspCheckStatus" value="init int-app-api-meta process-rtsp-check-status close"/>
        <property name="StatusCorrection"
                  value="init int-app-api perform-status-correction close"/>
        <property name="GetProgramTokenAvailability" value="init get-program-token-availability close"/>
        <property name="GetBankTokenAvailability" value="init get-program-token-availability close"/>
        <property name="GetUserStatusWithCentralMapper"
                  value="init int-app-api-meta parse-req-get-add-request close"/>
        <property name="PCBDCDisbursementUser"
                  value="init int-app-api save-db disburse-pbt close"/>
        <property name="PCBDCReturnExpired"
                  value="init int-app-api parse-pbt-return-expired-request pcbdc-redeem-expired close"/>
        <property name="ValidatePayee" value="init int-app-api-meta validate-payee close"/>
        <!--<property name="VerifyWalletPin" value="init verify-pin close"/>-->

        <property name="Merchant.CheckTxnStatus"
                  value="init int-app-api-meta load-merchant-customer find-merchant-txn close"/>
        <property name="ReqUpdateToken" value="init int-app-api-meta request-update-token close"/>
        <property name="MobileWhitelist" value="init mobile-whitelist close"/>

        <!-- PCBDC APIs -->
        <property name="CreatePCBDCSponsor"
                  value="init int-app-api-meta create-pcbdc-sponsor close"/>


        <property name="CreatePCBDCRule"
                  value="init int-app-api-meta parse-pcbdc-rule-request check-pcbdc-sponsor-cbs-action prepare-ref-lookup-request select-validation-service-endpoint route-to-endpoint create-pcbdc-rule split-sponsor-requisition-requests save-db prepare-rule-creation-response close"/>

        <property name="UpdatePCBDCRule"
                  value="init int-app-api-meta parse-pcbdc-rule-request prepare-ref-lookup-request select-validation-service-endpoint route-to-endpoint create-pcbdc-rule prepare-rule-creation-response close"/>

        <!--        Duplicate Transaction with different ITC (identification purpose)   -->
        <property name="UpdatePCBDCRuleMeta"
                  value="init int-app-api-meta parse-pcbdc-rule-request prepare-ref-lookup-request select-validation-service-endpoint route-to-endpoint create-pcbdc-rule prepare-rule-creation-response close"/>

        <property name="EnhancePCBDCRule"
                  value="init int-app-api-meta parse-pcbdc-rule-enhance-request check-pcbdc-sponsor-cbs-action split-sponsor-requisition-requests save-db prepare-rule-creation-response close"/>

        <property name="CheckPCBDCConversionStatus"
                  value="init int-app-api-meta check-sponsor-requisition-status close"/>

        <property name="PCBDCSponsorRequisition"
                  value="int-app-api parse-sponsor-requisition-request handle-requisition-request update-sponsor-requisition-status save-db close"/>

        <property name="FetchPCBDCRuleDetails"
                  value="prepare-req-view-ref-lookup select-validation-service-endpoint route-to-endpoint save-rule-data save-db close"/>

        <property name="unknown" value="notsupported close"/>
    </participant>

    <group name="prepare-req-view-ref-lookup">
        <participant class="com.sarvatra.rtsp.transaction.PrepareReqViewRefLookup" realm="prepare-req-view-ref-lookup"
                     logger="Q2">
        </participant>
    </group>

    <group name="save-rule-data">
        <participant class="com.sarvatra.rtsp.transaction.SaveSponsorRule" realm="save-rule-data"
                     logger="Q2">
        </participant>
        &normalize;
    </group>

    <group name="parse-pcbdc-rule-request">
        <participant class="com.sarvatra.rtsp.transaction.ParsePCBDCRuleCreationRequest"
                     realm="parse-rule-creation-request" logger="Q2">
        </participant>
    </group>

    <group name="parse-pbt-return-expired-request">
        <participant logger="Q2" class="com.sarvatra.rtsp.transaction.ParsePBTExpiryRequest"
                     realm="parse-pbt-expiry-request">
        </participant>
    </group>

    <group name="prepare-ref-lookup-request">
        <participant logger="Q2" class="com.sarvatra.rtsp.transaction.PrepareRefLookups"
                     realm="prepare-ref-lookup-request">
        </participant>
    </group>

    <group name="check-pcbdc-sponsor-cbs-action">
        <participant class="org.jpos.transaction.participant.Switch" logger="Q2"
                     realm="decide-requisition-request-flow">
            <property name="txnname" value="PCBDC_SPONSOR_DEBIT"/>
            <property name="true" value="debit-bank-account"/>
            <property name="false" value=""/>
            <property name="unknown" value=""/>
        </participant>
    </group>

    <group name="parse-pcbdc-rule-enhance-request">
        <participant logger="Q2" class="com.sarvatra.rtsp.transaction.ParseEnhanceRuleRequest"
                     realm="parse-rule-creation-request">
        </participant>
    </group>

    <group name="create-pcbdc-rule">
        <participant logger="Q2" class="com.sarvatra.rtsp.transaction.ParseValidationServiceResponse"
                     realm="parse-rule-creation-response">
        </participant>
        <participant class="org.jpos.transaction.participant.Switch" logger="Q2"
                     realm="decide-requisition-request-flow">
            <property name="txnname" value="PCBDC_SPONSOR_DEBIT_REVERSAL"/>
            <property name="true" value="debit-reversal"/>
            <property name="false" value=""/>
            <property name="unknown" value=""/>
        </participant>
        <participant logger="Q2" class="com.sarvatra.rtsp.transaction.CreatePCBDCRuleTomas"
                     realm="create-pcbdc-rule-tomas">
        </participant>
    </group>

    <group name="split-sponsor-requisition-requests">
        <participant logger="Q2" class="com.sarvatra.rtsp.transaction.CreateSponsorRequisitionRequests"
                     realm="split-sponsor-requisition-requests">
        </participant>
    </group>

    <group name="prepare-rule-creation-response">
        <participant logger="Q2" class="com.sarvatra.rtsp.transaction.PreparePCBDCRuleCreationResponse"
                     realm="create-pcbdc-rule">
        </participant>
        &normalize;
    </group>

    <group name="parse-sponsor-requisition-request">
        <participant logger="Q2" class="com.sarvatra.rtsp.transaction.ParseSponsorRequisitionRequest"
                     realm="parse-sponsor-requisition-request">
        </participant>
    </group>

    <group name="handle-requisition-request">
        <participant class="org.jpos.transaction.participant.Switch" logger="Q2"
                     realm="decide-requisition-request-flow">
            <property name="txnname" value="REQUISITION_REQUEST_STATUS"/>
            <property name="REQUESTED" value="parse-issue-token"/>
            <property name="PENDING" value="parse-issue-token"/>
            <property name="FAILED" value="parse-issue-token"/>
            <property name="ALLOCATED" value="confirm-tomas-txn"/>
            <property name="CONFIRM_TIMEOUT" value="confirm-tomas-txn"/>
            <property name="unknown" value=""/>
        </participant>
    </group>

    <group name="process-rtsp-check-status">
        <participant logger="Q2" class="com.sarvatra.rtsp.transaction.ops.ProcessRtspCheckStatus"
                     realm="rtsp-check-status">
        </participant>
        &normalize;
    </group>

    <group name="perform-status-correction">
        <participant logger="Q2" class="com.sarvatra.rtsp.transaction.ops.ProcessRtspCheckStatus"
                     realm="rtsp-check-status(status-correction)">
        </participant>
        <participant class="org.jpos.transaction.participant.Switch" logger="Q2" realm="check-status-resolution">
            <property name="txnname" value="RESOLVE_STATUS"/>
            <property name="CONFIRM" value="confirm-tomas-txn"/>
            <property name="CANCEL" value="confirm-tomas-txn"/>
            <property name="CANCEL_LOAD" value="confirm-tomas-txn cbs-debit-reversal"/>
            <property name="ISSUE" value="force-issue-token"/>
            <property name="unknown" value=""/>
        </participant>
        <!--                <participant class="com.sarvatra.rtsp.transaction.tomas.ConfirmTxn"-->
        <!--                             realm="confirm-tomas-txn(status-correction)"-->
        <!--                             logger="Q2">-->
        <!--                </participant>-->
        <participant class="com.sarvatra.rtsp.transaction.ops.PostProcessingStatusCorrection"
                     realm="post-processing(status-correction)"
                     logger="Q2">
        </participant>

        &normalize;
    </group>

    <group name="force-issue-token">
        <participant class="com.sarvatra.rtsp.transaction.tomas.IssueToken" realm="force-issue-token"
                     configuration-factory="org.jpos.util.GenericConfigurationFactory"
                     logger="Q2">
            <property name="valid-cbs-rc" value="env:VALID_CBS_RC"/>
        </participant>
    </group>

    <group name="update-sponsor-requisition-status">
        <participant logger="Q2" class="com.sarvatra.rtsp.transaction.UpdateSponsorRequisitionStatus"
                     realm="update-sponsor-requisition-request">
        </participant>
    </group>

    <group name="update-pcbdc-status-for-abort">
        <participant logger="Q2" class="com.sarvatra.rtsp.transaction.UpdatePcbdcStatusForAbort"
                     realm="update-pcbdc-status-for-abort">
        </participant>
    </group>

    <group name="check-sponsor-requisition-status">
        <participant class="com.sarvatra.rtsp.transaction.CheckSponsorRequisitionStatus" logger="Q2"
                     realm="check-individual-requisition-requests"/>
        &normalize;
    </group>

    <group name="select-validation-service-endpoint">
        <participant class="com.sarvatra.rtsp.transaction.SetTargetEndpoint" logger="Q2" realm="set-pso-endpoint">
            <property name="endpoint" value="VALIDATION-SERVICE"/>
        </participant>
        <participant class="com.sarvatra.rtsp.transaction.SignValidationServicePayload" logger="Q2"
                     realm="sign-validation-service-payload">
        </participant>
    </group>

    <group name="create-pcbdc-sponsor">
        <participant class="com.sarvatra.rtsp.transaction.users.CheckDuplicateSponsor"
                     logger="Q2"
                     realm="check-duplicate-sponsor"/>
        <participant class="com.sarvatra.rtsp.transaction.users.CreateSponsor"
                     logger="Q2"
                     realm="create-sponsor"/>
        <participant class="com.sarvatra.rtsp.transaction.users.CreateSponsorCustomer"
                     logger="Q2"
                     realm="create-sponsor-customer"/>
        <participant class="com.sarvatra.rtsp.transaction.tomas.CreateWallet" realm="tomas-open-wallet"
                     logger="Q2">
        </participant>
        <participant class="com.sarvatra.rtsp.transaction.users.CreateSponsorWalletDetails"
                     logger="Q2"
                     realm="create-sponsor-wallet-details"/>
        <participant class="org.jpos.transaction.participant.Switch" logger="Q2" realm="sponsor-account-creation-check">
            <property name="txnname" value="SPONSOR_ACCOUNT_CREATION"/>
            <property name="true" value="create-sponsor-account"/>
            <property name="false" value=""/>
            <property name="" value=""/>
        </participant>
        <participant class="com.sarvatra.rtsp.transaction.users.CreateSponsorMerchant"
                     logger="Q2"
                     realm="create-sponsor-customer-account"/>
        <participant class="com.sarvatra.rtsp.transaction.merchant.GetMerchantRefId" logger="Q2"
                     realm="refid-merchant" configuration-factory="org.jpos.util.GenericConfigurationFactory">
            <property name="big.base.url" value="env:BIG_BASE_URL"/>
            <property name="big.api.url" value="/verify-card-details"/>
            <property name="http-client" value="BIG-MDM"/>
        </participant>
        <participant class="com.sarvatra.rtsp.transaction.users.UpdateSponsorDetails"
                     logger="Q2"
                     realm="update-sponsor-details"/>
        &normalize;
    </group>

    <group name="create-sponsor-account">
        <participant class="com.sarvatra.rtsp.transaction.users.CreateSponsorCustomerAccount"
                     logger="Q2"
                     realm="create-sponsor-customer-account"/>
    </group>

    <group name="mobile-whitelist">
        <participant logger="Q2" class="com.sarvatra.rtsp.transaction.CustomerMobileWhitelist" realm="mobile-whitelist">
        </participant>
        &normalize;
    </group>

    <group name="load-customer-wallet">
        <participant class="com.sarvatra.rtsp.transaction.users.VerifyActiveWallet" realm="find-customer-wallet"
                     logger="Q2">
        </participant>
        &normalize;
    </group>

    <group name="prepare-wallet-response">
        <participant class="com.sarvatra.rtsp.transaction.users.PrepareWalletResponse" realm="prepare-wallet-response"
                     logger="Q2">
        </participant>
        &normalize;
    </group>

    <group name="prepare-response-external-load">
        <participant class="com.sarvatra.rtsp.transaction.users.PrepareResponseExternalLoad"
                     realm="prepare-response-external-load"
                     logger="Q2">
        </participant>
        &normalize;
    </group>

    <group name="fetch-token-wallet">
        <participant class="com.sarvatra.rtsp.transaction.users.FetchWalletLoad" realm="prepare-wallet-response"
                     logger="Q2">
        </participant>
        &normalize;
    </group>

    <group name="check-rules">
        <participant class="com.sarvatra.rtsp.transaction.users.CheckProfileRuleApplicableContext"
                     realm="check-cooling-period"
                     logger="Q2">
        </participant>
        <participant class="com.sarvatra.rtsp.transaction.users.CheckPcbdcNoKycTransaction"
                     realm="check-pcbdc-no-kyc-txn"
                     logger="Q2">
            <property name="pcbdc-no-kyc-itc"
                      value="PCBDCDisbursementUser,PSO.ReqPay.DEBIT.PAY,PSO.ReqPay.CREDIT.PAY,PSO.ReqPay.DEBIT,PSO.ReqPay.CREDIT"/>
        </participant>
        <participant class="com.sarvatra.rtsp.transaction.sre.PrepareForRuleCheck" realm="prepare-for-check"
                     logger="Q2">
        </participant>

        &check-risk-lite;
    </group>
    <group name="pcbdc-redeem-expired">
        <participant class="com.sarvatra.rtsp.transaction.tomas.PcbdcRedeemExpired" realm="pcbdc-redeem-expired"
                     logger="Q2">
        </participant>
        <participant class="org.jpos.transaction.participant.Switch" logger="Q2" realm="check-payment-mode">
            <property name="txnname" value="KCC_TRANSACTION"/>
            <property name="true" value="credit-bank-account-kcc"/>
            <property name="false" value=""/>
            <property name="unknown" value=""/>
        </participant>

        <participant class="com.sarvatra.rtsp.transaction.tomas.ConfirmTxn"
                     realm="confirm-tomas-txn(pcbdc-user-return-expired)"
                     logger="Q2">
        </participant>
        <participant class="com.sarvatra.rtsp.transaction.tomas.ConfirmTxn"
                     realm="confirm-tomas-txn(pcbdc-user-return-expired-abort-flow)"
                     logger="Q2">
            <property name="abort-flow-only" value="true"/>
        </participant>
        <participant class="com.sarvatra.rtsp.pbt.ValidateUserExpiryStatus"
                     realm="validate-user-expiry-status"
                     logger="Q2">
        </participant>
        &normalize;
    </group>
    <group name="get-program-token-availability">
        <participant class="com.sarvatra.rtsp.transaction.GetTokenAvailability"
                     realm="get-token-availability"
                     logger="Q2"/>
        &normalize;
    </group>
    <group name="parse-issue-token">
        <participant class="com.sarvatra.rtsp.transaction.CheckBlockedCustomer" logger="Q2"
                     realm="check-blocked-customer">
            <property name="PAYEE" value="MOBILE,DEVICE,VPA,WALLET"/>
        </participant>
        <participant class="org.jpos.transaction.participant.Switch" logger="Q2" realm="check-payment-mode">
            <property name="txnname" value="PAYMENT_MODE"/>
            <property name="DEBIT" value="find-account"/>
            <property name="UPI" value=""/>
            <property name="SPONSOR_REQUISITION" value=""/>
            <property name="unknown" value=""/>
        </participant>

        <participant class="com.sarvatra.rtsp.transaction.tokens.InitIssueToken" realm="init-issue-token"
                     logger="Q2">
        </participant>

        <participant class="org.jpos.transaction.participant.Switch" logger="Q2" realm="check-payment-mode">
            <property name="txnname" value="PAYMENT_MODE"/>
            <property name="DEBIT" value="debit-bank-account issue-token"/>
            <property name="UPI" value="await-upi-callback"/>
            <property name="SPONSOR_REQUISITION" value="pcbdc-issue-token"/>
            <property name="unknown" value="debit-bank-account issue-token"/>
        </participant>
    </group>

    <group name="find-account">
        <participant class="com.sarvatra.rtsp.transaction.accounts.FindAccount" realm="find-customer-account"
                     logger="Q2">
            <property name="find-onus-account" value="true"/>
        </participant>
    </group>

    <group name="debit-bank-account">
        <participant class="com.sarvatra.rtsp.transaction.accounts.SetCbsCallData" realm="set-cbs-call-data(debit)"
                     logger="Q2">
            <property name="force-onus-only" value="true"/>
        </participant>

        &onus_bank_debit_account;
    </group>

    <group name="debit-bank-account-kcc">
        <participant class="com.sarvatra.rtsp.transaction.accounts.SetCbsCallData" realm="set-cbs-call-data(debit)"
                     logger="Q2">
            <property name="force-onus-only" value="true"/>
        </participant>

        &onus_bank_debit_account;

        <participant class="com.sarvatra.rtsp.transaction.ValidateKccCBSResponse" realm="validate-cbs-leg"
                     logger="Q2">
        </participant>
    </group>

    <group name="credit-bank-account-kcc">
        <participant class="com.sarvatra.rtsp.transaction.accounts.SetCbsCallData" realm="set-cbs-call-data(debit)"
                     logger="Q2">
            <property name="force-onus-only" value="true"/>
        </participant>

        &onus_bank_credit_account;

        <participant class="com.sarvatra.rtsp.transaction.ValidateKccCBSResponse" realm="validate-cbs-leg"
                     logger="Q2">
        </participant>
    </group>
    
    <group name="disburse-pbt">
        <participant class="com.sarvatra.rtsp.transaction.ParsePBTDisbursalRequest"
                     realm="parse-disbursal-request"
                     logger="Q2">
        </participant>

        <!-- Save DB -->
        <participant class="org.jpos.transaction.SaveDB" realm="save-db" logger="Q2"
                     configuration-factory="org.jpos.util.GenericConfigurationFactory">
            <property name="timeout" value="env:DB_OPEN_TIMEOUT:300"/>
        </participant>

        <!-- Check status with tomas -->
        <participant class="org.jpos.transaction.participant.Switch" logger="Q2"
                     realm="decide-pcbdc-disbursement-action">
            <property name="txnname" value="CHECK_TOMAS_TXN"/>
            <property name="true" value="check-pcbdc-disbursal-status"/>
            <property name="false" value=""/>
            <property name="unknown" value=""/>
        </participant>
        <!-- Cancel original transaction -->
        <participant class="org.jpos.transaction.participant.Switch" logger="Q2"
                     realm="decide-pcbdc-disbursement-action">
            <property name="txnname" value="CANCEL_ORIGINAL_PCBDC_DISBURSEMENT"/>
            <property name="true" value="confirm-tomas-txn"/>
            <property name="false" value=""/>
            <property name="unknown" value=""/>
        </participant>
        <!-- Decide action for disbursement -->
        <participant class="org.jpos.transaction.participant.Switch" logger="Q2"
                     realm="decide-pcbdc-disbursement-action">
            <property name="txnname" value="PROCESS_PCBDC_DISBURSEMENT"/>
            <property name="true" value="process-pbt-disbursal update-pcbdc-status-for-abort"/>
            <property name="false" value=""/>
            <property name="unknown" value="process-pbt-disbursal update-pcbdc-status-for-abort"/>
        </participant>
        &normalize;
    </group>

    <group name="check-pcbdc-disbursal-status">
        <participant class="com.sarvatra.rtsp.transaction.CheckPCBDCDisbursalStatus" logger="Q2"
                     realm="check-pcbdc-disbursal-status">
        </participant>
    </group>

    <group name="process-pbt-disbursal">
        <participant class="com.sarvatra.rtsp.transaction.ValidateKccTransaction" realm="validate-kcc-transaction"
                     logger="Q2">
            <property name="check-kcc-account-debit-in-pcbdc-issue" value="true"/>
            <property name="ignore-legacy-flow" value="false"/>
        </participant>
        <participant class="org.jpos.transaction.participant.Switch" logger="Q2" realm="process-req-pay">
            <property name="txnname" value="KCC_ACCOUNT_ACTION"/>
            <property name="KCC_DEBIT" value="debit-bank-account-kcc"/>
            <property name="KCC_DEBIT_REVERSAL" value="cbs-debit-reversal"/>
            <property name="unknown" value=""/>
        </participant>

        <participant class="org.jpos.transaction.participant.Switch" logger="Q2" realm="Switch">
            <property name="txnname" value="CHECK_RULES_PCBDC"/>
            <property name="true" value="check-rules"/>
            <property name="false" value=""/>
        </participant>
        <participant class="org.jpos.transaction.participant.Switch" logger="Q2" realm="switch-disburse-type">
            <property name="txnname" value="PAYEE_FLOW"/>
            <property name="ONUS_PCBDC_ENABLED" value="process-merchant-payout-request"/>
            <property name="ONUS_PCBDC_DISABLED" value="process-merchant-payout-request"/>
            <property name="ONUS_NO_PCBDC_WALLET" value="process-merchant-payout-request"/>
            <property name="OFFUS_BANK_PCBDC_ENABLED" value="process-merchant-payout-request"/>
            <property name="OFFUS_BANK_PCBDC_DISABLED" value="process-merchant-payout-request"/>
        </participant>
        &normalize;

    </group>
    <group name="populate-intermediate-response">
        <participant class="com.sarvatra.rtsp.transaction.SetTxnResponseCode" logger="Q2" realm="set-txn-response-code">
            <property name="rc" value="0091"></property>
            <property name="commit-phase" value="false"></property>
        </participant>

        <participant class="org.jpos.transaction.participant.Switch" logger="Q2" realm="decide-intermediate-response">
            <property name="txnname" value="RC"/>
            <property name="0091" value=""/>
            <property name="0000" value=""/>
            <property name="00ZA" value=""/>
            <property name="duplicate.transaction" value="ignore-duplicate"/>
            <property name="unknown" value="prepare-failure-response select-app-endpoint route-to-endpoint"/>
        </participant>
    </group>

    <group name="prepare-failure-response">
        &normalize;

        <participant class="org.jpos.transaction.participant.Switch" logger="Q2" realm="decide-pay-success-flow">
            <property name="txnname" value="TXNNAME"/>
            <property name="APP.ReqPay.LOAD" value="transfer-failure"/>
            <property name="MerchantPayout.POOL" value="transfer-failure"/>
            <property name="MerchantCommonPay.LOAD" value="transfer-failure"/>
            <property name="APP.ReqPay.PAY" value="transfer-failure-common"/>
            <property name="APP.ReqPay" value="transfer-failure"/>
            <property name="APP.ReqUserDeReg" value="de-reg-failure"/>
            <property name="unknown" value="unknown-response"/>
        </participant>
    </group>

    <group name="transfer-failure">
        <participant class="org.jpos.transaction.participant.Switch" logger="Q2" realm="process-req-pay">
            <property name="txnname" value="PAY_TYPE"/>
            <property name="COLLECT" value="debit-reversal transfer-failure-common "/>
            <property name="unknown" value="debit-reversal transfer-failure-common"/>
        </participant>
    </group>

    <group name="debit-reversal">
        <participant class="com.sarvatra.rtsp.transaction.accounts.PrepareForReversal" logger="Q2"
                     realm="prepare-for-reversal">
        </participant>

        <participant class="org.jpos.transaction.participant.Switch" logger="Q2" realm="decide-tomas-reversal">
            <property name="txnname" value="REVERSE_TOMAS_TXN"/>
            <property name="true" value="tomas-reversal"/>
            <property name="unknown" value=""/>
        </participant>

        <participant class="org.jpos.transaction.participant.Switch" logger="Q2" realm="decide-cbs-reversal">
            <property name="txnname" value="REVERSE_CBS_TXN"/>
            <property name="true" value="cbs-debit-reversal"/>
            <property name="unknown" value=""/>
        </participant>

        <participant class="org.jpos.transaction.participant.Switch" logger="Q2" realm="decide-upi-reversal">
            <property name="txnname" value="REVERSE_UPI_TXN"/>
            <property name="true" value="upi-debit-reversal"/>
            <property name="unknown" value=""/>
        </participant>
    </group>
    <group name="cbs-debit-reversal">
        &debit_reversal_request;
    </group>
    <group name="transfer-failure-common">
        <participant class="com.sarvatra.rtsp.transaction.transfer.app.PrepareFinalRespPay" realm="common-failure"
                     logger="Q2">
        </participant>
    </group>

    <group name="find-merchant-txn">
        <participant class="com.sarvatra.rtsp.transaction.merchant.FindMerchantTxn" logger="Q2"
                     realm="find-merchant-txn">
        </participant>

        &normalize;

        <participant class="com.sarvatra.rtsp.transaction.merchant.PrepareMerchantCheckTxnResponse" logger="Q2"
                     realm="prepare-check-txn-resp">
            <property file="cfg/pso-rc.cfg"/>
        </participant>
    </group>

    <group name="int-app-api">
        <participant class="com.sarvatra.rtsp.transaction.CreateFinancialTranlog" realm="create-fin-tranlog"
                     logger="Q2">
        </participant>

    </group>

    <group name="check-duplicate">
        <participant class="com.sarvatra.rtsp.transaction.CheckDuplicate" realm="check-duplicate"
                     logger="Q2">
        </participant>
    </group>

    <group name="init">
        <participant class="com.sarvatra.rtsp.transaction.ValidateRequest" logger="Q2" realm="validate-request"
                     configuration-factory="org.jpos.util.GenericConfigurationFactory">
            <property name="ignore.hmac.failure" value="env:IGNORE_CHANNEL_HMAC_FAILURE:true"/>
        </participant>
    </group>

    <group name="parse-manual-redeem-request">
        <participant class="com.sarvatra.rtsp.transaction.ParseManualRedeemRequest" logger="Q2"
                     realm="parse-manual-redeem-request">
        </participant>
    </group>
    <group name="parse-req-get-add-request">
        <participant class="com.sarvatra.rtsp.transaction.ParseReqGetAddRequest" logger="Q2"
                     realm="parse-req-get-add-request">
        </participant>
        <participant class="org.jpos.transaction.participant.Switch" logger="Q2" realm="decide-validate-flow">
            <property name="txnname" value="PAYEE_FLOW"/>
            <property name="VALIDATE_CENTRAL_MAPPER"
                      value="save-db proceed-with-req-get-add select-pso-endpoint route-to-endpoint"/>
            <property name="VALIDATE_PCBDC_BANK_ACCOUNT" value="fetch-bank-kcc-account"/>
        </participant>
    </group>
    <group name="fetch-bank-kcc-account">
        &onus_bank_fetch_account;

        <participant class="com.sarvatra.rtsp.transaction.accounts.FetchPcbdcKccAccounts"
                     realm="log-pcbdc-fetched-accounts"
                     logger="Q2">
            <property name="mask-account-part-length" value="4"/>
        </participant>

        <participant class="org.jpos.transaction.participant.Switch" logger="Q2" realm="decide-validate-flow">
            <property name="txnname" value="CREATE_KCC_NO_KYC_WALLET"/>
            <property name="CREATE_WALLET_WITH_KCC_ACC" value="create-kcc-no-kyc-wallet"/>
            <property name="CHECK_KCC_ACC_LINK" value="check-kcc-link-account"/>
        </participant>

    </group>

    <group name="check-kcc-link-account">
        <participant class="com.sarvatra.rtsp.transaction.accounts.LogFetchedPcbdcKccAccounts"
                     realm="log-pcbdc-fetched-accounts"
                     logger="Q2">
            <property name="mask-account-part-length" value="4"/>
        </participant>
    </group>
    <group name="create-kcc-no-kyc-wallet">
        <participant class="com.sarvatra.rtsp.transaction.registration.CreateNoKycWallet" realm="create-no-kyc-wallet"
                     logger="Q2">
        </participant>

        <participant class="com.sarvatra.rtsp.transaction.tomas.CreateWallet" realm="tomas-open-wallet"
                     logger="Q2">
        </participant>

        <participant class="com.sarvatra.rtsp.transaction.accounts.LogFetchedPcbdcKccAccounts"
                     realm="log-pcbdc-fetched-accounts"
                     logger="Q2">
            <property name="mask-account-part-length" value="4"/>
        </participant>
    </group>

    <group name="handle-redeem-debit">
        <participant class="com.sarvatra.rtsp.transaction.tomas.BalanceInquiry" realm="tomas-bi"
                     logger="Q2">
        </participant>
        <participant class="com.sarvatra.rtsp.transaction.ValidateManualRedeemAmount"
                     realm="validate-manual-redeem-amount"
                     logger="Q2">
        </participant>
        <participant class="com.sarvatra.rtsp.transaction.tomas.RedeemToken" realm="redeem-token"
                     logger="Q2">
        </participant>
        &onus_merchant_bank_credit_account;
        <participant class="com.sarvatra.rtsp.transaction.IgnoreCbsError" realm="ignore-cbs-error"
                     logger="Q2">
        </participant>
        &normalize;
        <participant class="com.sarvatra.rtsp.transaction.LogManualRedeemStatus" logger="Q2"
                     realm="log-manual-redeem-status">
        </participant>
        <participant class="com.sarvatra.rtsp.transaction.tomas.ConfirmTxn"
                     realm="confirm-tomas-txn(merchant-manual-redeem)"
                     logger="Q2">
        </participant>
        &normalize;
    </group>

    <group name="process-payout-request">
        <participant class="com.sarvatra.rtsp.transaction.ValidateRefundTransactionConfig"
                     realm="validate-merchant-payout"
                     logger="Q2">
        </participant>

        <participant class="org.jpos.transaction.participant.Switch" logger="Q2" realm="validate-pin-flag">
            <property name="txnname" value="VERIFY_PIN"/>
            <property name="Y" value="parse-request-creds check-wallet-pin"/>
        </participant>


        <participant class="org.jpos.transaction.participant.Switch" logger="Q2" realm="refund-auto-load-flag">
            <property name="txnname" value="REFUND_AUTO_LOAD_FROM_ACC"/>
            <property name="Y" value="validate-wallet-balance-with-txn-amt"/>
            <property name="N" value=""/>
            <property name="unknown" value=""/>
        </participant>

        <participant class="org.jpos.transaction.participant.Switch" logger="Q2" realm="decide-validate-flow">
            <property name="txnname" value="PAYEE_FLOW"/>
            <property name="ONUS" value="onus-validate-vpa-payout"/>
            <property name="VALIDATE_CENTRAL_MAPPER"
                      value="proceed-with-req-get-add select-pso-endpoint route-to-endpoint validate-offus-va-resp"/>
            <property name="REGULAR_VPA"
                      value="offus-validate-vpa save-db select-pso-endpoint route-to-endpoint validate-offus-va-resp"/>
            <property name="UPI_VPA"
                      value="validate-vpa-with-upi save-db validate-vpa-response"/>
        </participant>


        <participant class="org.jpos.transaction.participant.Switch" logger="Q2" realm="check-allow-req-pay">
            <property name="txnname" value="PAYEE_ACTION"/>
            <property name="ONUS" value="onus-validate-vpa-payout process-merchant-flow"/>
            <property name="OUTWARD" value="process-merchant-flow"/>
            <property name="DIRECT_LOAD" value="process-merchant-flow"/>
            <property name="RESOLVED" value="process-merchant-flow"/>
            <property name="UPI_VPA" value="process-merchant-flow"/>
            <property name="ABORT" value=""/>
        </participant>

        <participant class="com.sarvatra.rtsp.transaction.PostMerchantPayoutResponse" realm="resp-merchant-payout"
                     logger="Q2">
        </participant>

        &normalize;
    </group>
    <group name="update-final-status">

        <participant class="com.sarvatra.rtsp.transaction.UpdateResponseForMerchantTxnPool"
                     realm="update-response-for-merchantTxnPool"
                     logger="Q2">
        </participant>

    </group>

    <group name="process-upi-payout-request">
        <participant class="com.sarvatra.rtsp.transaction.VerifyAdjustmentType"
                     realm="verify-request"
                     logger="Q2">
        </participant>

        <participant class="org.jpos.transaction.participant.Switch" logger="Q2" realm="check-adjustment-type">
            <property name="txnname" value="ADJUSTMENT_TYPE"/>
            <property name="RET" value="confirm-tomas-txn"/>
            <property name="UPI_FAILURE" value="confirm-tomas-txn"/>
            <property name="UPI_NA" value="confirm-tomas-txn"/>
            <property name="UPI_SUCCESS" value="confirm-tomas-txn"/>
            <property name="TCC" value="confirm-tomas-txn"/>
            <property name="Debit Reversal Confirmation" value="confirm-tomas-txn"/>
            <property name="Chargeback Acceptance" value="issue-token"/>
            <property name="Credit Adjustment" value="issue-token"/>
            <property name="Fraud Chargeback Accept" value="issue-token"/>
            <property name="RTSP_FAILURE" value="issue-token"/>
            <property name="RTSP_NA" value="issue-token"/>
            <property name="Wrong Credit Chargeback Acceptance" value="issue-token"/>
        </participant>

        <participant class="com.sarvatra.rtsp.transaction.PopulateAdjustmentRC" realm="populate-rc"
                     logger="Q2">
            <property name="tomas-retry-rc" value=",8000,8002,8003,8006,8012,8013,8015,8031,8049,"/>
            <property name="tomas-fail-rc-tl-success" value=",8023,8043,8044,"/>
        </participant>

        &normalize;

        <participant class="org.jpos.transaction.participant.Switch" logger="Q2" realm="check-adjustment-type">
            <property name="txnname" value="ADJUSTMENT_TYPE"/>
            <property name="Compensation" value="issue-token"/>
        </participant>

        &normalize;

    </group>
    <group name="validate-vpa-with-upi">
        &upi-validate-vpa;
    </group>
    <group name="validate-vpa-response">
        &normalize;

        <participant class="com.sarvatra.rtsp.transaction.txn.PrepareRespValAdd" realm="resp-val-add"
                     logger="Q2">
        </participant>
    </group>
    <group name="confirm-tomas-txn">
        <participant class="com.sarvatra.rtsp.transaction.tomas.ConfirmTxn"
                     realm="confirm-tomas-txn" logger="Q2">
        </participant>
    </group>

    <group name="pcbdc-issue-token">
        <participant class="com.sarvatra.rtsp.transaction.tomas.IssueToken" realm="issue-token"
                     configuration-factory="org.jpos.util.GenericConfigurationFactory"
                     logger="Q2">
            <property name="valid-cbs-rc" value="env:VALID_CBS_RC"/>
        </participant>

        &normalize;
    </group>

    <group name="issue-token">

        <participant class="com.sarvatra.rtsp.transaction.tomas.IssueToken" realm="issue-token"
                     configuration-factory="org.jpos.util.GenericConfigurationFactory"
                     logger="Q2">
            <property name="valid-cbs-rc" value="env:VALID_CBS_RC"/>
        </participant>

        &normalize;

        <participant class="org.jpos.transaction.participant.Switch" logger="Q2" realm="check-notify-issue-token">
            <property name="txnname" value="TOMAS_RESPONSE_RC"/>
            <property name="0000" value="notify-issue-token select-app-endpoint route-to-endpoint"/>
            <property name="unknown" value=""/>
        </participant>
    </group>


    <group name="verify-pin">
        <participant class="com.sarvatra.rtsp.transaction.PrepareCustomerPinVerificationReq"
                     realm="customer-pin-verification"
                     logger="Q2">
        </participant>

        <participant class="org.jpos.transaction.participant.Switch" logger="Q2" realm="validate-pin-flag">
            <property name="txnname" value="VERIFY_PIN"/>
            <property name="Y" value="parse-request-creds check-wallet-pin"/>
        </participant>


        <participant class="org.jpos.transaction.participant.Switch" logger="Q2" realm="check-tomas-wallet-creation">
            <property name="txnname" value="VALID_PIN"/>
            <property name="true" value="verify-pin-success"/>
        </participant>

        &normalize;

    </group>

    <group name="tomas-confirm">
        <participant class="com.sarvatra.rtsp.transaction.tomas.ReversalTxn" realm="reversal-txn"
                     logger="Q2">
        </participant>
    </group>


    <group name="verify-pin-success">
        <participant class="com.sarvatra.rtsp.transaction.PopulateContextProperties" realm="populate-properties"
                     logger="Q2">
            <property name="resolve-field" value="RC"/>
            <property name="source.RC" value="0000"/>
        </participant>
    </group>

    <group name="parse-request-creds">
        <participant class="com.sarvatra.rtsp.transaction.users.ParseCredBlocks" realm="parse-req-block"
                     logger="Q2">
            <property name="key.PIN.NTPIN" value="RAW_NEW_PIN"/>
            <property name="key.PIN.TPIN" value="RAW_MPIN"/>
            <property name="key.PIN.OTP" value="BANK_OTP"/>
            <property name="key.CARD.DEBITCARD" value="CARD_DATA"/>

        </participant>

        <participant class="com.sarvatra.rtsp.transaction.security.BaseCredentialsDecryptEngine"
                     realm="decrypt-cred-block" logger="Q2"
                     configuration-factory="org.jpos.util.GenericConfigurationFactory">
            <property name="secure-fields" value="RAW_NEW_PIN,BANK_OTP,CARD_DATA,RAW_MPIN,TokenSecret"/>
            <property name="trace" value="env:ENABLE_TRACE:false"/>
        </participant>

        <participant class="com.sarvatra.rtsp.transaction.cl.DecodeCredBlocks" realm="decode-cl-blocks" logger="Q2"
                     configuration-factory="org.jpos.util.GenericConfigurationFactory">
            <property name="block" value="CARD_DATA"/>
            <property name="block" value="PSO"/>

            <property name="type.CARD_DATA" value="CARD"/>
            <property name="type.PSO" value="PSO"/>
            <property name="trace" value="env:ENABLE_TRACE:false"/>
        </participant>
    </group>

    <group name="check-wallet-pin">
        <participant class="com.sarvatra.rtsp.transaction.users.CheckPIN" realm="check-wallet-pin" logger="Q2">
            <property name="pin-key" value="RTSP_MPIN"/>
        </participant>
    </group>

    <group name="update-reward">
        <participant class="com.sarvatra.rtsp.transaction.UpdateReward" realm="update-reward" logger="Q2">
        </participant>
    </group>

    <group name="process-merchant-flow">
        <participant class="org.jpos.transaction.participant.Switch" logger="Q2" realm="process-req-pay">
            <property name="txnname" value="PAY_TYPE"/>
            <property name="REFUND" value="process-merchant-refund-request"/>
            <property name="PAYOUT" value="process-merchant-payout-request"/>
            <property name="DBT" value="process-merchant-payout-request"/>
            <property name="CASHBACK" value="process-merchant-payout-request"/>
            <property name="LOAD" value="process-merchant-load-request populate-intermediate-response"/>
        </participant>
    </group>

    <group name="onus-validate-vpa-payout">
        <participant class="com.sarvatra.rtsp.transaction.txn.ValidateAddress" realm="validate-address"
                     logger="Q2">
        </participant>
        <participant class="com.sarvatra.rtsp.transaction.ValidatePayeeVpaOnMerchantPay"
                     realm="validate-address-merchant-pay"
                     logger="Q2">
        </participant>

    </group>
    <group name="validateva-response-poll">
        <participant class="com.sarvatra.rtsp.transaction.MerchantPayValidateVaAsyncWait"
                     realm="validate-address-resp-wait"
                     logger="Q2">
        </participant>

    </group>

    <group name="validate-offus-va-resp-for-pcbdc">
        <participant class="com.sarvatra.rtsp.transaction.txn.outward.ValidateOffusAddress" realm="offus-val-add-resp"
                     logger="Q2">
        </participant>
    </group>

    <group name="validate-offus-va-resp">
        <participant class="com.sarvatra.rtsp.transaction.txn.outward.ValidateOffusAddress" realm="offus-val-add-resp"
                     logger="Q2">
        </participant>
        <participant class="com.sarvatra.rtsp.transaction.ValidatePayeeVpaOnMerchantPay"
                     realm="validate-address-merchant-pay-offus"
                     logger="Q2">
        </participant>
    </group>

    <group name="delay">
        <participant class="org.jpos.transaction.participant.AbortedDelay" realm="delay">
            <property name="prepare-delay" value="1"></property>
            <property name="commit-delay" value="1"></property>
            <property name="abort-delay" value="1"></property>
        </participant>
    </group>
    <group name="offus-validate-vpa">
        <!--  <participant class="com.sarvatra.rtsp.transaction.async.CreateAsyncRecordCache"
                       realm="create-async-record-cache" logger="Q2">

          </participant>
  -->

        <participant class="com.sarvatra.rtsp.transaction.txn.outward.PrepareReqValAdd" realm="prepare-req-val-add"
                     logger="Q2">
        </participant>
    </group>

    <group name="proceed-with-req-get-add">
        <participant class="com.sarvatra.rtsp.transaction.registration.PrepareReqGetAdd" realm="prepare-response"
                     logger="Q2">
        </participant>
    </group>

    <group name="select-pso-endpoint">
        <participant class="com.sarvatra.rtsp.transaction.SetTargetEndpoint" logger="Q2" realm="set-pso-endpoint">
            <property name="endpoint" value="PSO"/>
        </participant>
    </group>

    <group name="process-merchant-refund-request">

        <participant class="com.sarvatra.rtsp.transaction.tomas.TransferTokenRefund" realm="transfer-token"
                     logger="Q2">
        </participant>

        <participant class="com.sarvatra.rtsp.transaction.PrepareMerchantRefundRequest" realm="parse-refund-request"
                     logger="Q2">
        </participant>

        <participant class="org.jpos.transaction.participant.Switch" logger="Q2" realm="check-notify-transfer-token">
            <property name="txnname" value="TOMAS_RESPONSE_RC"/>
            <property name="0000" value="notify-transfer-token select-pso-fin-endpoint route-to-endpoint"/>
            <property name="unknown" value=""/>
        </participant>
    </group>

    <group name="save-db">
        <participant class="org.jpos.transaction.SaveDB" realm="save-db" logger="Q2"
                     configuration-factory="org.jpos.util.GenericConfigurationFactory">
            <property name="timeout" value="env:DB_OPEN_TIMEOUT:300"/>
        </participant>
    </group>

    <group name="validate-payee">
        <participant class="com.sarvatra.rtsp.transaction.merchant.ValidatePayeeRequest"
                     realm="validate-payee-address"
                     logger="Q2">
        </participant>
        <!--
                <participant class="org.jpos.transaction.participant.Switch" logger="Q2" realm="decide-validate-flow">
                    <property name="txnname" value="PAYEE_FLOW"/>
                    <property name="ONUS" value="onus-validate-vpa-payout"/>
                    <property name="VALIDATE_CENTRAL_MAPPER"
                              value="proceed-with-req-get-add select-pso-endpoint route-to-endpoint"/>
                    <property name="REGULAR_VPA"
                              value="offus-validate-vpa select-pso-endpoint route-to-endpoint validateva-response-poll"/>
                </participant>-->

        <participant class="com.sarvatra.rtsp.transaction.merchant.PostValidatePayeeResponse" logger="Q2"
                     realm="post-payee-va-response">
        </participant>

        &normalize;

    </group>
    <group name="select-pso-fin-endpoint">
        <participant class="com.sarvatra.rtsp.transaction.SetTargetEndpoint" logger="Q2" realm="set-pso-endpoint">
            <property name="endpoint" value="PSO-FIN"/>
        </participant>
    </group>

    <group name="route-to-endpoint">
        <participant class="com.sarvatra.rtsp.transaction.PrepareXml" logger="Q2" realm="prepare-xml">
        </participant>

        <participant class="com.sarvatra.rtsp.transaction.SignPayload" logger="Q2" realm="sign-payload">
        </participant>

        <participant class="com.sarvatra.rtsp.transaction.QueryHttp" logger="Q2" realm="query-http">
            <property name="header" value="x-olympic"/>
            <property name="header.x-olympic" value="joJFwcBPPl4MiYpLiQ4iGVWjIzkAsbRy"/>

            <property name="header" value="User-Agent"/>
            <property name="header.User-Agent" value="PostmanRuntime/7.28.4"/>
        </participant>
    </group>

    <group name="process-merchant-payout-request">
        <participant class="org.jpos.transaction.participant.Switch" logger="Q2" realm="check-transfer-token">
            <property name="txnname" value="UPI_TRANSACTION"/>
            <property name="true" value="redeem-token-merchant"/>
            <property name="false" value="transfer-token-merchant"/>
            <property name="unknown" value="transfer-token-merchant"/>
        </participant>

        <participant class="com.sarvatra.rtsp.transaction.PrepareMerchantPayoutRequest" realm="parse-refund-request"
                     logger="Q2">
        </participant>

        <participant class="org.jpos.transaction.participant.Switch" logger="Q2" realm="check-notify-transfer-token">
            <property name="txnname" value="TOMAS_RESPONSE_RC"/>
            <property name="0000" value="save-db notify-transfer-token select-pso-fin-endpoint route-to-endpoint"/>
            <property name="unknown" value=""/>
        </participant>
    </group>

    <group name="transfer-token-merchant">
        <participant class="com.sarvatra.rtsp.transaction.tomas.TransferTokenRefund" realm="transfer-token-merchant"
                     logger="Q2">
        </participant>

        <participant class="org.jpos.transaction.participant.Switch" logger="Q2" realm="process-req-pay">
            <property name="txnname" value="KCC_ACCOUNT_ACTION"/>
            <property name="KCC_DEBIT_REVERSAL" value="cbs-debit-reversal"/>
            <property name="unknown" value=""/>
        </participant>
    </group>


    <group name="redeem-token-merchant">
        <participant class="com.sarvatra.rtsp.transaction.tomas.RedeemToken" realm="redeem-token-merchant"
                     logger="Q2">
        </participant>
    </group>

    <group name="merchant-location-finder">
        <participant class="com.sarvatra.rtsp.transaction.merchant.MerchantLocationFinder" logger="Q2"
                     realm="get-merchant-location-list">

        </participant>

        &normalize;
    </group>

    <group name="merchant-log-intent">
        <participant class="com.sarvatra.rtsp.transaction.merchant.MerchantLogIntent" logger="Q2"
                     realm="merchant-log-intent">

        </participant>
        <participant class="com.sarvatra.rtsp.transaction.merchant.PostLogIntent" realm="post-log-intent"
                     logger="Q2">

        </participant>

        &normalize;
    </group>
    <group name="process-merchant-load-request">

        <participant class="com.sarvatra.rtsp.transaction.PrepareMerchantLoadRequest" realm="prepare-load-account"
                     logger="Q2">
        </participant>

        <participant class="com.sarvatra.rtsp.transaction.accounts.FindAccount" realm="find-customer-account"
                     logger="Q2">
            <property name="find-onus-account" value="true"/>
        </participant>

        <participant class="com.sarvatra.rtsp.transaction.tokens.InitIssueToken" realm="init-issue-token"
                     logger="Q2">
        </participant>

        <participant class="com.sarvatra.rtsp.transaction.accounts.SetCbsCallData" realm="set-cbs-call-data(debit)"
                     logger="Q2">
            <property name="force-onus-only" value="true"/>
        </participant>

        <participant class="org.jpos.transaction.participant.Switch" logger="Q2" realm="check-skip-cbs-leg-debit">
            <property name="txnname" value="SKIP_CBS"/>
            <property name="true" value=""/>
            <property name="false" value="cbs-debit-leg"/>
        </participant>

        <participant class="com.sarvatra.rtsp.transaction.tomas.IssueToken" realm="issue-token"
                     configuration-factory="org.jpos.util.GenericConfigurationFactory"
                     logger="Q2">
            <property name="valid-cbs-rc" value="env:VALID_CBS_RC"/>
        </participant>

        &normalize;

        <participant class="org.jpos.transaction.participant.Switch" logger="Q2" realm="check-notify-issue-token">
            <property name="txnname" value="TOMAS_RESPONSE_RC"/>
            <property name="0000" value="notify-issue-token"/>
            <property name="unknown" value=""/>
        </participant>

    </group>

    <group name="cbs-debit-leg">
        &onus_bank_debit_account;
    </group>

    <group name="notify-issue-token">
        <participant class="com.sarvatra.rtsp.transaction.transfer.app.PrepareFinalRespPay" realm="prepare-issue-token"
                     logger="Q2">
        </participant>

        <participant class="com.sarvatra.rtsp.transaction.transfer.QueueUpdateDtsp" realm="update-central-provider"
                     logger="Q2">
            <property name="txn-manager-queue" value="RTSP.DTSP.TXN"/>
            <property name="tl-key" value="TOKEN_TRANLOG"/>
            <property name="target-pojo-key" value="POJO"/>
            <property name="tomas-resp-key" value="TOMAS_RESPONSE"/>
        </participant>
    </group>

    <group name="select-app-endpoint">
        <participant class="com.sarvatra.rtsp.transaction.SetTargetEndpoint" logger="Q2" realm="set-app-endpoint">
            <property name="endpoint" value="APP"/>
        </participant>
    </group>

    <group name="int-app-api-meta">
        <participant class="com.sarvatra.rtsp.transaction.CreateMetaTranlog" realm="create-meta-tranlog-meta"
                     logger="Q2">
        </participant>
    </group>

    <group name="customer-invitee">
        <participant class="com.sarvatra.rtsp.transaction.CustomerInvitee" realm="customer-invitee" logger="Q2">
        </participant>

        &normalize;
    </group>

    <group name="merchant-statement">
        <participant class="com.sarvatra.rtsp.transaction.merchant.PreTxnDetails" logger="Q2" realm="pre-txn-details">
        </participant>

        <participant class="com.sarvatra.rtsp.transaction.txn.GetAllTxn" realm="find-customer-txn" logger="Q2">
            <property name="set-success" value="false"/>
        </participant>

        <participant class="com.sarvatra.rtsp.transaction.merchant.PostTxnDetails" logger="Q2"
                     configuration-factory="org.jpos.util.GenericConfigurationFactory"
                     realm="post-txn-details">
            <property name="cig-helper-clazz.merchant-statement-details"
                      value="env:CIG_HELPER_MERCHANT_STATEMENT_DETAILS"/>
        </participant>
        &normalize;
    </group>

    <group name="merchant-settlement-statement">
        <participant class="com.sarvatra.rtsp.transaction.merchant.SettlementPreTxnDetails" logger="Q2"
                     realm="pre-settlement-txn-details">
        </participant>

        <participant class="com.sarvatra.rtsp.transaction.txn.GetAllSettlementTxn" realm="find-customer-settlement-txn"
                     logger="Q2">
        </participant>

        <participant class="com.sarvatra.rtsp.transaction.merchant.PostTxnSettlementDetails" logger="Q2"
                     configuration-factory="org.jpos.util.GenericConfigurationFactory"
                     realm="post-settlement-txn-details">
        </participant>
        &normalize;
    </group>

    <group name="get-settlement-details">
        <participant class="com.sarvatra.rtsp.transaction.merchant.SettledPreTxnDetails" logger="Q2"
                     realm="pre-settlement-txn-details">
        </participant>

        <participant class="com.sarvatra.rtsp.transaction.txn.GetAllSettledTxn" realm="find-customer-total-txn"
                     logger="Q2">

        </participant>

        <participant class="com.sarvatra.rtsp.transaction.merchant.MerchantSettlementDetails" logger="Q2"
                     configuration-factory="org.jpos.util.GenericConfigurationFactory"
                     realm="post-settled-txn-details">
        </participant>
        &normalize;
    </group>

    <group name="load-merchant-customer">
        <participant class="com.sarvatra.rtsp.transaction.merchant.LoadMerchantCustomer" logger="Q2"
                     realm="pre-balance">
        </participant>
    </group>

    <group name="merchant-details">
        <participant class="com.sarvatra.rtsp.transaction.merchant.LoadAndPostMerchantDetails" logger="Q2"
                     configuration-factory="org.jpos.util.GenericConfigurationFactory"
                     realm="get-merchant-details">
            <property name="cig-helper-clazz.merchant-details" value="env:CIG_HELPER_MERCHANT_DETAILS"/>
        </participant>

        &normalize;
    </group>

    <group name="merchant-status">
        <participant class="com.sarvatra.rtsp.transaction.merchant.LoadAndPostMerchantDetailsStatus" logger="Q2"
                     realm="check-merchant-status">
        </participant>
        &normalize;
    </group>

    <group name="kyc-limit-config">
        <participant class="com.sarvatra.rtsp.transaction.merchant.PrepareKycBasedLimitConfiguration" logger="Q2"
                     realm="kyc-based-limit">
            <property name="min-kyc-config-limit" value="100"/>
            <property name="kyc-config-merchant-limit" value="100000"/>
        </participant>
        &normalize;
    </group>

    <group name="merchant-balance">
        <participant class="com.sarvatra.rtsp.transaction.tomas.BalanceInquiry" realm="tomas.balance" logger="Q2">
            <property name="set-success" value="false"/>
        </participant>

        <participant class="org.jpos.transaction.participant.Switch" logger="Q2" realm="check-tomas-wallet-creation">
            <property name="txnname" value="SECONDARY_WALLET_IDENTIFIER"/>
            <property name="Y" value="fetch-secondary-wallet-balance"/>
        </participant>

        <participant class="com.sarvatra.rtsp.transaction.merchant.PostBalance" logger="Q2" realm="post-balance">
        </participant>
        &normalize;
    </group>

    <group name="fetch-secondary-wallet-balance">
        <participant class="com.sarvatra.rtsp.transaction.tomas.BalanceInquiry" realm="tomas.balance" logger="Q2">
            <property name="set-success" value="false"/>
            <property name="secondary-wallet-balance-call" value="true"/>
        </participant>
    </group>
    <group name="ensure-merchant-details">
        <participant class="org.jpos.transaction.participant.Switch" logger="Q2" realm="check-tomas-wallet-creation">
            <property name="txnname" value="MERCHANT_ACTION"/>
            <property name="UPDATE" value="reject-duplicate-merchant"/>
            <property name="CREATE" value="create-merchant-details create-tomas-wallet create-ux-user"/>
        </participant>

        &normalize;
    </group>

    <group name="create-merchant">
        <participant class="org.jpos.transaction.participant.Switch" logger="Q2" realm="check-tomas-wallet-creation">
            <property name="txnname" value="MERCHANT_ACTION"/>
            <property name="UPDATE" value="reject-duplicate-merchant"/>
            <property name="CREATE" value="create-merchant-details create-tomas-wallet create-ux-user"/>
        </participant>

        &normalize;
    </group>

    <group name="update-merchant">
        <participant class="org.jpos.transaction.participant.Switch" logger="Q2" realm="check-tomas-wallet-creation">
            <property name="txnname" value="MERCHANT_ACTION"/>
            <property name="UPDATE" value="create-merchant-details create-tomas-wallet create-ux-user"/>
            <property name="CREATE" value="reject-not-found-merchant"/>
        </participant>

        &normalize;
    </group>

    <group name="reject-duplicate-merchant">
        <participant class="com.sarvatra.rtsp.transaction.PopulateContextProperties" realm="populate-properties"
                     logger="Q2">
            <property name="resolve-field" value="RC"/>
            <property name="source.RC" value="duplicate.merchant.mobile"/>
        </participant>
    </group>

    <group name="reject-not-found-merchant">
        <participant class="com.sarvatra.rtsp.transaction.PopulateContextProperties" realm="populate-properties"
                     logger="Q2">
            <property name="resolve-field" value="RC"/>
            <property name="source.RC" value="merchant.not.found"/>
        </participant>
    </group>

    <group name="create-merchant-details">
        <participant class="com.sarvatra.rtsp.transaction.merchant.EnsureMerchantCustomer" logger="Q2"
                     realm="ensure-merchant">
        </participant>

        <participant class="com.sarvatra.rtsp.transaction.merchant.OnboardMerchant" logger="Q2"
                     realm="onbaord-merchant">
        </participant>

        <participant class="com.sarvatra.rtsp.transaction.merchant.GetMerchantRefId" logger="Q2"
                     realm="refid-merchant" configuration-factory="org.jpos.util.GenericConfigurationFactory">
            <property name="big.base.url" value="env:BIG_BASE_URL"/>
            <property name="big.api.url" value="/verify-card-details"/>
            <property name="http-client" value="BIG-MD"/>
        </participant>

        <participant class="com.sarvatra.rtsp.pilot.transaction.ModifyMerchantPilotCount"
                     realm="modify-merchant-pilot-count"
                     logger="Q2">
        </participant>
    </group>

    <group name="create-tomas-wallet">
        <participant class="com.sarvatra.rtsp.transaction.tomas.CreateWallet" realm="tomas-open-wallet"
                     logger="Q2">
        </participant>
    </group>

    <group name="request-update-token">
        <participant class="com.sarvatra.rtsp.transaction.ops.ReqUpdateToken" realm="ux-req-update-token"
                     logger="Q2">
        </participant>
        &normalize;
    </group>

    <group name="create-ux-user">
        <participant class="com.sarvatra.rtsp.transaction.merchant.CreateMerchantEEUser" realm="merchant-ee-user"
                     logger="Q2">
        </participant>
    </group>

    <group name="check-merchant">
        <participant class="com.sarvatra.rtsp.transaction.merchant.CheckMerchant" logger="Q2" realm="check-merchant">
        </participant>
    </group>

    <group name="sms-notify">
        &sms-notification;
    </group>

    <group name="close">

        <participant class="com.sarvatra.rtsp.transaction.LogCleanup" logger="Q2" realm="clean-up">
        </participant>

        <participant class="org.jpos.transaction.Close" logger="Q2" realm="close">
            <property name="checkpoint" value="close"/>
        </participant>

        <participant class="com.sarvatra.rtsp.transaction.PrepareResponse" logger="Q2" realm="prepare-response">
        </participant>


        <participant class="com.sarvatra.rtsp.transaction.ReleaseLocks" logger="Q2" realm="release-locks">
        </participant>

        &queue-sms-notification;

    </group>

    <group name="notsupported">
        <participant class="org.jpos.jpts.UnsupportedTxn" logger="Q2" realm="unsupported"/>
    </group>

    <participant class="org.jpos.jpts.ProtectDebugInfo" logger="Q2" realm="protect-debug-info">
        <property name="fields" value="2,14,35,36,45,48,52,54,62,96,102,103,121"/>
        <property name="xml-messages" value="XML_REQUEST,FETCHED_CUSTOMER_ACCOUNTS,APP_CARD_DATA,TXN_FOLLOWUP"/>
        <property name="wipe-context-keys" value="POJO,FETCHED_CUSTOMER_ACCOUNTS,MOBILE,TXN_STMT,TOMAS_TOKENS,TOMAS_RESPONSE"/>
        <property name="xml-clear-element-xpath"
                  value="/Link,/Resp/RegIdDetails/Id,/ReqDetails/User/DeviceInfo,/Payees/Payee/Device,/Payer/Device,/PayerRegIdDetails/Id,/ReqDetails/User/Creds/Cred/Datacode,/ReqDetails/User/Creds/Cred/Data,/ReqDetails/Payer/Creds/Cred/Datacode,/ReqDetails/Payer/Creds/Cred/Data,/ReqDetails/Payee/Creds/Cred/Datacode,/ResDetails/User/Creds/Cred/Datacode,/ResDetails/Payer/Creds/Cred/Datacode,/ResDetails/Creds/Cred/Datacode,/Payer/Creds/Cred/Data"/>
        <property name="json-payload" value="POJO,USER_INFO"/>
        <property name="enable-sensitive-info" value="true"/>
    </participant>

</rtsp-config-txnmgr>



