# Diagnostic Services

The `vectornav` driver exposes three operator-triggered services for runtime
maintenance of the VectorNav sensor. They are independent of the automatic
reconnect timer and only run when invoked.

All services use the standard `std_srvs/srv/Trigger` type. Service names are
relative to the node, so the default fully qualified names are:

- `/vectornav/reset_device`
- `/vectornav/tare_device`
- `/vectornav/reset_acc_bias`

## Services

### reset_device

- Type: `std_srvs/srv/Trigger`
- SDK call: `vn::sensors::VnSensor::reset()`
- Effect: sends a soft reset command. The sensor reboots and the serial link
  drops momentarily. The driver's `reconnect_ms` timer will re-establish the
  connection on its own.
- Success message: `Device reset command sent. Wait for device to reboot.`

### tare_device

- Type: `std_srvs/srv/Trigger`
- SDK call: `vn::sensors::VnSensor::tare()`
- Effect: zeros the current attitude. The current orientation becomes the new
  reference frame for yaw/pitch/roll output.
- Success message: `Device tare command sent. Current readings zeroed.`

### reset_acc_bias

- Type: `std_srvs/srv/Trigger`
- SDK calls:
  - `writeAccelerationCompensation(identity_gain, {0,0,0}, true)`
  - `writeSettings(true)` (persists to flash)
- Effect: clears the accelerometer compensation register (identity gain,
  zero bias) and saves the change. A device reset is required for the new
  bias to take effect.
- Success message: `Bias register set to zero. Please reset the device to apply.`
- Implementation note: the handler holds an internal mutex
  (`service_acc_bias_mtx_`) so concurrent calls serialize the flash write.

## Example invocations

```
ros2 service call /vectornav/reset_device std_srvs/srv/Trigger {}
ros2 service call /vectornav/tare_device std_srvs/srv/Trigger {}
ros2 service call /vectornav/reset_acc_bias std_srvs/srv/Trigger {}
```

A successful call returns:

```
success: True
message: "<one of the strings above>"
```

## Failure behavior

Each handler wraps its SDK call in a `try/catch`. If the sensor handle is null,
or the SDK throws (no device, comms timeout, unsupported command for the
device family), the response is:

```
success: False
message: "<exception text from the SDK>"
```

The node does not crash, and the next call can be attempted after the
condition is resolved (typically when the reconnect timer succeeds).

## Disabling the services

The services are advertised by default. To disable, set the parameter at
launch time:

```
ros2 run vectornav vectornav --ros-args -p enable_diagnostic_services:=false
```

When disabled, the three services are simply not created; calls will fail
with `service not available`.

## Source attribution

The design and method names follow
[RobotnikAutomation/vectornav](https://github.com/RobotnikAutomation/vectornav),
specifically the change titled `feat: add services to reset and tare device (#21)`.
The snslab port keeps the same service names and response strings, but adapts
to the snslab class layout: the sensor handle is `std::shared_ptr<vn::sensors::VnSensor>`
(`vs_->...`) and the services are gated behind the `enable_diagnostic_services`
parameter.
