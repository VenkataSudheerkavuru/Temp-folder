
public class ReportFactory {
    public static Report getReportGenerator(ReportType reportType) {
        switch (reportType) {
            case DAILY:
                return new DailyPerformanceReport();
        }
    }
}

Interface
---------------
public interface Report {
    File generateReport(SearchRequest request);
  }

Controller
-----------------
@RequestMapping(value = "getReport",method = RequestMethod.POST)
File getReport(@RequestBody SearchRequest searchRequest);

Impl
----------
@Service
public class ReportServiceImpl {
    private final ReportFactory reportFactory;

    public File getReport(ReportRequest request) {
        //for any kind of Validations
        validateRequest(request);
        // Get appropriate report generator (daily , monthly…)
        Report reportGenerator = reportFactory.getReportGenerator(request.getReportType());
        // Generate report
        return reportGenerator.generateReport(request);
    }
}

Dailyreport.class
----------------
@Component
public class DailyReport implements Report {
    @Override
    public File generateReport(ReportRequest request) {
        // Daily specific implementation
    }
}
Monthly.class
-------------
@Component
public class MonthlyReport implements Report {
    @Override
    public File generateReport(ReportRequest request) {
        // Monthly specific implementation
    }
}

