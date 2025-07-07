ReportType.java
public enum ReportType {
    DAILY,
    MONTHLY
}

// Report.java
public interface Report {
    byte[] generateReport(ReportRequest request);  // returns byte[] for sync, uploadable for async
    ReportType getType();
}


---

// DailyReport.java
@Component
public class DailyReport implements Report {
    @Override
    public byte[] generateReport(ReportRequest request) {
        // actual Excel generation logic here
        return "Daily Report".getBytes(); // mock byte[]
    }

    @Override
    public ReportType getType() {
        return ReportType.DAILY;
    }
}

// MonthlyReport.java
@Component
public class MonthlyReport implements Report {
    @Override
    public byte[] generateReport(ReportRequest request) {
        return "Monthly Report".getBytes(); // mock byte[]
    }

    @Override
    public ReportType getType() {
        return ReportType.MONTHLY;
    }
}


---

// ReportFactory.java
@Component
public class ReportFactory {

    private final Map<ReportType, Report> reportMap = new HashMap<>();

    @Autowired
    public ReportFactory(List<Report> reports) {
        for (Report report : reports) {
            reportMap.put(report.getType(), report);
        }
    }

    public Report getReportGenerator(ReportType type) {
        Report report = reportMap.get(type);
        if (report == null) {
            throw new IllegalArgumentException("No report found for type: " + type);
        }
        return report;
    }
}


---

// ReportRequest.java
public class ReportRequest {
    private ReportType reportType;
    private String fromDate;
    private String toDate;
    private String userId;
    // Getters and setters
}


---

// IReportController.java
@RequestMapping("/report")
public interface IReportController {

    @PostMapping("/getReport")
    ResponseEntity<byte[]> getReport(@RequestBody ReportRequest request);
}


---

// ReportControllerImpl.java
@RestController
public class ReportControllerImpl implements IReportController {

    private final ReportService reportService;

    @Autowired
    public ReportControllerImpl(ReportService reportService) {
        this.reportService = reportService;
    }

    @Override
    public ResponseEntity<byte[]> getReport(ReportRequest request) {
        byte[] data = reportService.getReport(request);
        return ResponseEntity.ok()
                .header(HttpHeaders.CONTENT_DISPOSITION, "attachment; filename=report.xlsx")
                .contentType(MediaType.APPLICATION_OCTET_STREAM)
                .body(data);
    }
}


---

// ReportService.java
public interface ReportService {
    byte[] getReport(ReportRequest request);
}


---

// ReportServiceImpl.java (for sync)
@Service
public class ReportServiceImpl implements ReportService {

    private final ReportFactory reportFactory;

    @Autowired
    public ReportServiceImpl(ReportFactory reportFactory) {
        this.reportFactory = reportFactory;
    }

    @Override
    public byte[] getReport(ReportRequest request) {
        validate(request);
        Report generator = reportFactory.getReportGenerator(request.getReportType());
        return generator.generateReport(request);
    }

    private void validate(ReportRequest request) {
        // Add your custom validations
    }
}


---

// AsyncReportProcessor.java
@Component
public class AsyncReportProcessor {

    @Autowired
    private ReportFactory reportFactory;

    @Autowired
    private S3UploaderService s3Uploader;

    public void processReportAsync(ReportRequest request) {
        Report report = reportFactory.getReportGenerator(request.getReportType());
        byte[] data = report.generateReport(request);
        String s3Url = s3Uploader.upload(data, "report_" + System.currentTimeMillis() + ".xlsx");
        System.out.println("Uploaded to S3: " + s3Url);
    }
}


---

// ReportScheduler.java
@Component
public class ReportScheduler {

    @Autowired
    private AsyncReportProcessor processor;

    @Scheduled(fixedRate = 120000) // every 2 minutes
    public void scheduleReports() {
        // Normally, fetch from DB where status = 'IN_QUEUE'
        ReportRequest req = new ReportRequest();
        req.setReportType(ReportType.DAILY);
        processor.processReportAsync(req);
    }
}


---

// S3UploaderService.java
@Service
public class S3UploaderService {

    public String upload(byte[] data, String filename) {
        // In real case, use AWS SDK here
        System.out.println("Simulated upload of " + filename);
        return "https://s3.aws.com/bucket/" + filename;
    }
}


