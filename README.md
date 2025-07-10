package in.securtime.core.processing.server.scheduling;


import in.securtime.core.employee.api.model.PreferredLanguage;
import in.securtime.core.notification.api.model.EmailInfo;
import in.securtime.core.notification.api.model.EmailRequestInformation;
import in.securtime.core.notification.api.model.SMSType;
import in.securtime.core.notification.api.model.SendSMSRequest;
import in.securtime.core.notification.client.EmailNotificationRestClient;
import in.securtime.core.notification.client.SMSNotificationRestClient;
import in.securtime.core.organization.api.model.OrganizationDetails;
import in.securtime.core.organization.client.OrganizationRestClient;
import in.securtime.core.processing.server.data.model.CommunicationStatus;
import in.securtime.core.processing.server.data.records.AttendanceApprovalNotification;
import in.securtime.core.processing.server.data.repository.AttendanceApprovalNotificationRepository;
import in.securtime.core.processing.server.models.AttendanceApprovalAlertModel;
import in.securtime.shared.util.date.DateUtil;
import in.securtime.shared.util.enums.Country;
import in.securtime.shared.util.enums.EmailLanguage;
import in.securtime.shared.util.model.SMSNumbetDetailsDTO;
import in.securtime.shared.util.model.SimpleResponse;
import in.securtime.shared.util.smsUtil.SmsUtil;
import org.json.JSONObject;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.core.io.Resource;
import org.springframework.http.HttpStatus;
import org.springframework.scheduling.annotation.Async;
import org.springframework.scheduling.annotation.EnableAsync;
import org.springframework.scheduling.concurrent.ThreadPoolTaskExecutor;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.io.IOException;
import java.time.LocalDate;
import java.time.YearMonth;
import java.util.ArrayList;
import java.util.List;
import java.util.Objects;
import java.util.concurrent.Executor;
import java.util.concurrent.ThreadPoolExecutor;

import static in.securtime.shared.util.emailUtil.EmailUtil.readFile;
import static in.securtime.shared.util.smsUtil.SmsUtil.safeReplace;

@Service
@EnableAsync
public class AsyncScheduledNotification {

    @Autowired
    private AttendanceApprovalNotificationRepository attendanceApprovalNotificationRepository;

    @Autowired
    private EmailNotificationRestClient emailNotificationRestClient;

    @Autowired
    private OrganizationRestClient organizationRestClient;

    @Autowired
    private SMSNotificationRestClient smsNotificationRestClient;
    
    @Value("classpath:smsContent.json")
    public Resource smsContent;

    @Value("${country:INDIA}")
    private String country;

    private static final org.slf4j.Logger LOGGER = LoggerFactory.getLogger(AsyncScheduledNotification.class);

    @Bean(name = "CustomAsyncExecutorForScheduledNotification")
    public Executor customThreadPoolTaskExecutor(){
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(5);
        executor.setMaxPoolSize(5);
        executor.setQueueCapacity(10);
        executor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
        executor.setThreadNamePrefix("Scheduled-Report -");
        executor.setWaitForTasksToCompleteOnShutdown(true);
        executor.initialize();
        return executor;
    }

    /**
     * Process a single email notification
     */
    @Transactional(readOnly = false)
    @Async("CustomAsyncExecutorForScheduledNotification")
    public void processSingleNotification(AttendanceApprovalNotification notification) {
        try {
            updateNotificationStatus(notification, CommunicationStatus.IN_PROGRESS);
            SimpleResponse simpleResponse = sendEmail(notification);

            if (simpleResponse.getStatus().equals(HttpStatus.OK)) {
                updateNotificationStatus(notification, CommunicationStatus.SUCCESS);
            } else {
                handleFailedNotification(notification, simpleResponse.getErrorMessage().get(0));
            }
        } catch (Exception e) {
            handleFailedNotification(notification, e.getMessage());
        }
    }

    /**
     * Process a single SMS notification
     */
    @Transactional(readOnly = false)
    @Async("CustomAsyncExecutorForScheduledNotification")
    public void processSingleSmsNotification(AttendanceApprovalNotification notification) {
        try {
            updateSmsNotificationStatus(notification, CommunicationStatus.IN_PROGRESS);
            SimpleResponse simpleResponse = sendSms(notification);

            if (simpleResponse.getStatus().equals(HttpStatus.OK)) {
                updateSmsNotificationStatus(notification, CommunicationStatus.SUCCESS);
            } else {
                handleFailedSmsNotification(notification, simpleResponse.getErrorMessage().get(0));
            }
        } catch (Exception e) {
            handleFailedSmsNotification(notification, e.getMessage());
        }
    }

    /**
     * Handle failed notification
     */
    private void handleFailedNotification(AttendanceApprovalNotification notification, String errorMessage) {
        notification.setEmailErrorMessage(errorMessage);
        updateNotificationStatus(notification, CommunicationStatus.FAILED);
    }

    /**
     * Update the status of the notification and save it
     */
    private void updateNotificationStatus(AttendanceApprovalNotification notification, CommunicationStatus status) {
        notification.setEmailStatus(status);
        attendanceApprovalNotificationRepository.save(notification);
    }


    /**
     * prepare and send emails
     */
    private SimpleResponse sendEmail(AttendanceApprovalNotification attendanceApprovalNotification) {

        EmailRequestInformation emailRequestInformation = prepareEmployeeCreationRequest(attendanceApprovalNotification);
        return emailNotificationRestClient.sendEmailLangSpecific(emailRequestInformation);
    }

    /**
     * prepare employee Creation request
     */
    private EmailRequestInformation prepareEmployeeCreationRequest(AttendanceApprovalNotification attendanceApprovalNotification) {

        EmailRequestInformation emailRequestInformation = new EmailRequestInformation();
        emailRequestInformation.setHandleBarData(prepareHandlerBarData(attendanceApprovalNotification));
        emailRequestInformation.setRecipient(prepareRecipientData(attendanceApprovalNotification));
        emailRequestInformation.setEmailType(attendanceApprovalNotification.getEmailType());
        return emailRequestInformation;
    }

    /**
     * prepare the recipient data
     */
    private List<EmailInfo> prepareRecipientData(AttendanceApprovalNotification attendanceApprovalNotification) {
        List<EmailInfo> recipientEmailInfo = new ArrayList<>();
        recipientEmailInfo.add(new EmailInfo(attendanceApprovalNotification.getEmailId(),
                getEmailLang(String.valueOf(attendanceApprovalNotification.getPreferredLanguage()))));
        return recipientEmailInfo;
    }


    /**
     * prepare the attendance approval model
     */
    private AttendanceApprovalAlertModel prepareHandlerBarData(AttendanceApprovalNotification attendanceApprovalNotification) {
        String orgUrl = prepareOrganizationUrl(attendanceApprovalNotification.getOrganizationId());

        // Get current year and month
        LocalDate currentDate = LocalDate.now();
        YearMonth currentYearMonth = YearMonth.from(currentDate);

        // Adjust the day to not exceed the month's length
        int adjustedDay = Math.min(attendanceApprovalNotification.getDefaultApprovalTime().intValue(), currentYearMonth.lengthOfMonth());

        // Create the date with the adjusted day
        LocalDate approvalLocalDate = currentDate.withDayOfMonth(adjustedDay);
        String approvalDate = DateUtil.formatDate(approvalLocalDate, DateUtil.DATEFORMAT_yyyyMMdd_slash);
        return new AttendanceApprovalAlertModel(attendanceApprovalNotification.getName(), approvalDate, orgUrl);
    }

    /**
     * prepare the organization url
     */
    private String prepareOrganizationUrl(Long organizationId) {
        OrganizationDetails organizationDetails = organizationRestClient.getOrganizationDetails(organizationId);
        return organizationDetails.getUrl();
    }

    /**
     * get the email language
     */
    public EmailLanguage getEmailLang(String lang) {
        return EmailLanguage.CHINESE.toString().equalsIgnoreCase(lang) ?
                EmailLanguage.CHINESE : EmailLanguage.ENGLISH;
    }

    

    /**
     * Handle failed SMS notification
     */
    private void handleFailedSmsNotification(AttendanceApprovalNotification notification, String errorMessage) {
        notification.setSmsErrorMessage(errorMessage);
        updateSmsNotificationStatus(notification, CommunicationStatus.FAILED);
    }

    /**
     * Update the status of the SMS notification and save it
     */
    private void updateSmsNotificationStatus(AttendanceApprovalNotification notification, CommunicationStatus status) {
        notification.setSmsStatus(status);
        attendanceApprovalNotificationRepository.save(notification);
    }

    /**
     * Prepare and send SMS
     */
    private SimpleResponse sendSms(AttendanceApprovalNotification attendanceApprovalNotification) {
        String smsContentString = fetchSmsContent(smsContent, attendanceApprovalNotification);
        if (!Objects.equals(attendanceApprovalNotification.getMobile(), "") && attendanceApprovalNotification.getMobile() != null)
            return sendSMSNotification(smsContentString, attendanceApprovalNotification.getMobile());
        throw new IllegalArgumentException("Mobile number is not provided for the notification.");
    }

    /**
     * send the sms notification
     */
    private SimpleResponse sendSMSNotification(String smsContentString, String mobile) {
        SendSMSRequest sendSMSRequest = prepareSMSRequest(smsContentString,mobile);
        return smsNotificationRestClient.sendSMS(sendSMSRequest);
    }

    /**
     * prepare the sms request
     */
    private SendSMSRequest prepareSMSRequest(String smsContentString, String mobile) {
        SendSMSRequest sendSMSRequest = new SendSMSRequest();
        SMSNumbetDetailsDTO smsNumbetDetailsDTO = SmsUtil.getNumberDetails(mobile, Country.valueOf(country.toUpperCase()));
        sendSMSRequest.setContent(smsContentString);
        sendSMSRequest.setNationCode(smsNumbetDetailsDTO.getCountryCode()!=null ? Integer.parseInt(smsNumbetDetailsDTO.getCountryCode()):0);
        sendSMSRequest.setPhoneNumber(smsNumbetDetailsDTO.getMobileNumber());
        return sendSMSRequest;
    }

    /**
     * fetch the sms content
     */
    private String fetchSmsContent(Resource smsContent, AttendanceApprovalNotification attendanceApprovalNotification) {
        String contentString = null;
        try {
            SMSType smsType = attendanceApprovalNotification.getSmsType();
            contentString = pickSutableSmsTemplate(smsType.toString(), attendanceApprovalNotification.getPreferredLanguage(),
                    smsContent, Country.valueOf(country.toUpperCase()));
            AttendanceApprovalAlertModel attendanceApprovalAlertModel = prepareHandlerBarData(attendanceApprovalNotification);
            contentString = replaceModelValueInContent(contentString, attendanceApprovalAlertModel);

        } catch (IOException e) {
            LOGGER.error("Error while picking teh sms content:" + e.getMessage());
        }
        return contentString;
    }

    /**
     * replace the model value in content
     *
     * @param contentString
     * @param attendanceApprovalAlertModel
     */
    private String replaceModelValueInContent(String contentString, AttendanceApprovalAlertModel attendanceApprovalAlertModel) {
        String output = null;
        if(contentString!=null) {
            output = contentString
                    .replace("{recipientName}", safeReplace(attendanceApprovalAlertModel.getRecipientName()))
                    .replace("{approvalDate}", safeReplace(attendanceApprovalAlertModel.getApprovalDate()))
                    .replace("{orgUrl}", safeReplace(attendanceApprovalAlertModel.getOrgUrl()));
        }
        return output;
    }

    /**
     * pickup the template based on language and sms Type
     *
     * @param smsType
     * @param preferredLanguage
     * @param smsContent
     * @param country
     * @return
     * @throws IOException
     */
    public String pickSutableSmsTemplate(String smsType, PreferredLanguage preferredLanguage, Resource smsContent, Country
            country) throws IOException {

        String content = null;
        JSONObject obj = readFile(smsContent);
        if (preferredLanguage == null) {
            preferredLanguage = getSmsLanguagebasedOnCountry(country);
        }
        JSONObject workflowObject = obj.optJSONObject(smsType);
        if (workflowObject != null) {
            content = workflowObject.optString(preferredLanguage.name(), null);
        }
        return content;
    }



    /**
     * get the email Language based on country
     *
     * @param country
     * @return
     */
    public PreferredLanguage getSmsLanguagebasedOnCountry(Country country){

        if(country.equals(Country.CHINA)){
            return PreferredLanguage.CHINESE;
        }else{
            return PreferredLanguage.ENGLISH;
        }
    }
}
