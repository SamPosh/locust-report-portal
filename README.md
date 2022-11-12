# locust-report-portal

This tutorial is about logging performance test results in Report Portal

First Initialize Report portal agent and create a launch and test
```
from reportportal_client import ReportPortalService

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

After each request is over below event will be triggered. It will be record the result in report portal logger.

```
@events.request.add_listener
def record_in_report_portal(request_type, name, response_time, response_length, response, context, exception, start_time, url, **kwargs):
    # We did log the results using logging.
    request_log= f"| Type: {request_type} | Request: {name} -> {url}| Response time: {response_time}ms | start_time: {start_date_time} |"
    service.log(time=timestamp(),
            message=request_log,
            level="INFO")
```

After all the execution is over ,check whether test is failed or passed

```
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
