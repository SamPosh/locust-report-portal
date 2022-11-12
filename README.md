# locust-report-portal

This tutorial is about logging performance test results in Report Portal.
It uses Report portal python client https://github.com/reportportal/client-Python

First Initialize Report portal agent and create a launch and test in test_start event
```
from locust import HttpUser, between, events
from reportportal_client import ReportPortalService
from locust.runners import WorkerRunner
import locust_plugins

@events.test_start.add_listener
def on_test_start(environment, **kwargs):
    # Initialize report portal service
    endpoint = "http://10.6.40.6:8080"
    project = "default"
    # You can get UUID from user profile page in the Report Portal.
    token = "1adf271d-505f-44a8-ad71-0afbdf8c83bd"
    launch_name = "Performance test"
    launch_doc = "Testing logging with attachment."
    global service
    service = ReportPortalService(endpoint=endpoint, project=project,
                                   token=token)
    # Start launch.
    global launch
    launch = service.start_launch(name=launch_name,
                              start_time=timestamp(),
                              description=launch_doc)
   
   # Start test item
   global item_id
   item_id = service.start_test_item(name="Test Case",
                                  description="First Test Case",
                                  start_time=timestamp(),
                                  item_type="STEP",
                                  parameters={"Testcase": "Testing some API performance",
                                              "key2": "val2"})
```

After each request is over request event will be triggered. In it request result will be recorded in report portal logs.
Note: As many requests will be recorded ,it is recorded in DEBUG level. Based on your project need you can modify it

```
@events.request.add_listener
def record_in_report_portal(request_type, name, response_time, response_length, response, context, exception, start_time, url, **kwargs):
    # We did log the results using logging.
    request_log= f"| Type: {request_type} | Request: {name} -> {url}| Response time: {response_time}ms | start_time: {start_date_time} |"
    service.log(time=timestamp(),
            message=request_log,
            level="DEBUG")
```

After all the execution is over ,check whether test is failed or passed

```
def log_stats_summary(environment, logger):
    """
        This will record the Result summary, percentile summary and Error summary in report portal
    """
    # Record result summary in Report portal
    service.log(time=timestamp(),
            message="Result Summary",
            level="INFO")
    summary = stats.get_stats_summary(environment.runner.stats, True)
    summary_str = '\n'.join([str(elem) for elem in summary])
    service.log(time=timestamp(),
            message=summary_str,
            level="INFO")
    
    # Record percentile summary in report portal
    percentile_stats = stats.get_percentile_stats_summary(environment.runner.stats)
    service.log(time=timestamp(),
            message="Percentile Summary",
            level="INFO")
    percentile_str = '\n'.join([str(elem) for elem in percentile_stats])
    service.log(time=timestamp(),
            message=percentile_str,
            level="INFO")
    
    # Record Error summary in report portal
    if len(environment.runner.stats.errors):
        service.log(time=timestamp(),
            message="Error Summary",
            level="INFO")
        # logger.info(stats.get_error_report_summary(environment.runner.stats))
        summary = stats.get_error_report_summary(environment.runner.stats)
        summary_str = '\n'.join([str(elem) for elem in summary])
        service.log(time=timestamp(),
            message=summary_str,
            level="INFO")
        
@events.quitting.add_listener
def do_checks(environment, **_kw):
  # Did check whether the tests pass or fail
    RP_TEST_STATUS = "PASSED"
    stats_total = environment.runner.stats.total
    fail_ratio = stats_total.fail_ratio

    check_fail_ratio = 0.01

    if fail_ratio > check_fail_ratio:
        error_msg=f"CHECK FAILED: fail ratio was {(fail_ratio*100):.2f}% (threshold {(check_fail_ratio*100):.2f}%)"
        service.log(time=timestamp(),
            message=error_msg,
            level="ERROR")
        
        RP_TEST_STATUS = "FAILED"
        environment.process_exit_code = 3
    else:
        success_msg=f"CHECK SUCCESSFUL: fail ratio was {(fail_ratio*100):.2f}% (threshold {(check_fail_ratio*100):.2f}%)"
        
        service.log(time=timestamp(),
            message=success_msg,
            level="INFO")


    service.finish_test_item(item_id=item_id, end_time=timestamp(), status=RP_TEST_STATUS)

    # Finish launch.
    service.finish_launch(end_time=timestamp())

    # Due to async nature of the service we need to call terminate() method which
    # ensures all pending requests to server are processed.
    # Failure to call terminate() may result in lost data.
    service.terminate()
```

Note: To observe performance test result with historical data, grafana dashboard would be better choice. But still reportportal would be required in some scenarios. Especially if your team sees all the test result from one single place , reportportal and it's widgets would be a good place.
