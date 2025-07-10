package in.securtime.core.processing.server.scheduling;

import in.securtime.core.processing.server.data.model.CommunicationStatus;
import in.securtime.core.processing.server.data.records.AttendanceApprovalNotification;
import in.securtime.core.processing.server.data.repository.AttendanceApprovalNotificationRepository;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.scheduling.annotation.EnableAsync;
import org.springframework.scheduling.annotation.EnableScheduling;
import org.springframework.scheduling.annotation.Scheduled;
import org.springframework.stereotype.Service;

import java.util.List;

@EnableScheduling
@EnableAsync
@Service
public class NotificationScheduler {



    @Autowired
    private AsyncScheduledNotification asyncScheduledNotification;

    @Autowired
    private AttendanceApprovalNotificationRepository attendanceApprovalNotificationRepository;


    /**
     * This method is scheduled to run every 2 minutes to fetch the top 5 records
     * from the AttendanceApprovalNotification table where the email status is IN_QUEUE.
     * It can be extended to process these records, such as sending emails.
     */
    @Scheduled(fixedRate = 120000) // every 2 min
    public void scheduleEmailNotifications() {
        List<AttendanceApprovalNotification> pendingEmailNotifications = fetchPendingEmailNotifications();
        if (!pendingEmailNotifications.isEmpty()) {
            processEmailNotifications(pendingEmailNotifications);
        }
        List<AttendanceApprovalNotification> pendingSmsNotifications = fetchPendingSmsNotifications();
        if (!pendingSmsNotifications.isEmpty()) {
            processSmsNotifications(pendingSmsNotifications);
        }
    }

    /**
     * Fetch the top 5 records from the AttendanceApprovalNotification table
     */
    private List<AttendanceApprovalNotification> fetchPendingEmailNotifications() {
        return attendanceApprovalNotificationRepository.findTop5ByEmailStatusOrderByCreationTimeAsc(CommunicationStatus.IN_QUEUE);
    }

    /**
     * Process the email notifications
     */
    private void processEmailNotifications(List<AttendanceApprovalNotification> notifications) {
        notifications.forEach(notification -> asyncScheduledNotification.processSingleNotification(notification));
    }


    /**
     * Fetch the top 5 records from the AttendanceApprovalNotification table for SMS
     */
    private List<AttendanceApprovalNotification> fetchPendingSmsNotifications() {
        return attendanceApprovalNotificationRepository.findTop5BySmsStatusOrderByCreationTimeAsc(CommunicationStatus.IN_QUEUE);
    }

    /**
     * Process the SMS notifications
     */
    private void processSmsNotifications(List<AttendanceApprovalNotification> notifications) {
        notifications.forEach(notification -> asyncScheduledNotification.processSingleSmsNotification(notification));
    }

}
