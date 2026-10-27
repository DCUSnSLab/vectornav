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
- SDK calls, in order:
  - `reset()`, then reopen the port at the sensor's saved baud rate
  - `writeAccelerationCompensation(identity_gain, {0,0,0}, true)`
  - `writeSettings(true)` (persists to flash)
  - `connect()` with the node parameters, which writes the driver configuration again
- Effect: clears the accelerometer compensation register (identity gain,
  zero bias) and saves it to flash. The new compensation applies as soon as
  it is written; no further reset is needed.
- Why the reset comes first: `writeSettings` saves every register, not only
  the one that changed. Without the reset it would also save the driver's
  runtime configuration (baud rate, binary outputs, ASCII output, sync
  control), and the sensor would boot with that configuration afterwards.
- Output stops for about 10 seconds while the sensor reboots and is
  reconfigured. The executor is blocked for that time.
- Success message: `Acceleration compensation cleared and saved.`

## Sensor lock

The SDK hands a response to whichever command is waiting for one; it does not
check which command the response belongs to. Two threads that send commands at
the same time can therefore receive each other's responses.

The magnetic calibration action runs in its own thread, so the node serializes
sensor commands with one mutex (`sensor_mtx_`):

| Caller | Behavior |
| --- | --- |
| `execute_cal` (mag cal thread) | holds the lock for the whole calibration |
| `reset_device`, `tare_device`, `reset_acc_bias` | return `success: False`, `Sensor busy (magnetic calibration in progress)` when the lock is held |
| `reconnect_timer` | skips the tick when the lock is held |
| `vel_aiding_cb` | drops the sample when the lock is held |
| `handle_cal_goal` | rejects a second calibration goal when the lock is held |

The executor side never waits for the lock, so a running calibration does not
block the executor.

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
The snslab port keeps the same service names, but adapts
to the snslab class layout: the sensor handle is `std::shared_ptr<vn::sensors::VnSensor>`
(`vs_->...`), the services are gated behind the `enable_diagnostic_services`
parameter, and `reset_acc_bias` reboots the sensor before saving (see above).
