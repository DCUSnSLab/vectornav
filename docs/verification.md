# Verification Plan — `feat/diagnostic-services`

This document defines how to verify the branch on a target machine that has
**ROS 2 Humble** installed and a **physical VectorNav sensor** connected. The
patches were authored on a Jazzy host without hardware; all runtime
acceptance criteria below are outstanding until executed there.

The branch adds three operator-triggered diagnostic services. Each has its
own test section with explicit PASS/FAIL criteria. The existing automatic
reconnect (`reconnect_ms`) is unchanged but is regression-tested at the end.

## Prerequisites

| Item | Required state |
|------|----------------|
| OS | Ubuntu 22.04 |
| ROS | Humble installed and sourced (`source /opt/ros/humble/setup.bash`) |
| Hardware | VectorNav VN-100 / VN-200 / VN-300 enumerated (e.g. `/dev/ttyUSB0`), powered |
| Workspace | `/home/nev/dev/ros2_ws` with `src/vectornav_snslab` checked out at `feat/diagnostic-services` |

Adjust `port`, `baud`, `frame_id` etc. via the existing config files. No
parameters were renamed by this branch.

## Step 1 — Build

```bash
cd /home/nev/dev/ros2_ws
rm -rf build/vectornav build/vectornav_msgs install/vectornav install/vectornav_msgs
colcon build --packages-select vectornav_msgs vectornav
```

**PASS:** both packages build without errors. The compile must include the
new sources without warnings on `std_srvs` linkage.

After the build:

```bash
source install/setup.bash
```

## Step 2 — Smoke test (device connected)

Goal: confirm the patched node still boots, connects, and publishes.

```bash
ros2 run vectornav vectornav
```

In another shell:

```bash
ros2 topic hz /vectornav/imu
ros2 topic echo /vectornav/imu --once
ros2 service list | grep vectornav
```

**PASS:**
- `/vectornav/imu` (and the other group topics: attitude, ins, gps if the
  device supports them) publish at the configured rate.
- The three new services appear:
  - `/vectornav/reset_device`
  - `/vectornav/tare_device`
  - `/vectornav/reset_acc_bias`
- No stack traces in the node log.

**FAIL:** node exits, no topics, services missing, or the existing
composable-node entry point is broken.

## Step 3 — `reset_device` (T1)

Goal: confirm the service triggers a real device reset and that the
existing reconnect timer brings the link back.

```bash
ros2 service call /vectornav/reset_device std_srvs/srv/Trigger {}
```

**Expected response:**

```
success: True
message: "Device reset command sent. Wait for device to reboot."
```

Then within seconds:

1. Topic publishing on `/vectornav/imu` pauses.
2. Node logs indicate a reconnect attempt (driven by `reconnect_ms`).
3. Within ≈1–3 seconds the device finishes rebooting and topics resume.

**PASS:** all three of the above hold.

**FAIL:**
- `success: False` with a non-`null` exception string while a sensor *is*
  connected — likely indicates the wrong SDK method was called.
- Topics never resume (reconnect timer regression — see Step 7).
- The node crashes during the reset.

## Step 4 — `tare_device` (T2)

Goal: confirm the service zeros the current attitude.

1. Place the device in a known orientation (level, facing arbitrary heading).
2. Call:

```bash
ros2 service call /vectornav/tare_device std_srvs/srv/Trigger {}
ros2 topic echo /vectornav/attitude --once
```

**Expected response:** `success: True`, message
`Device tare command sent. Current readings zeroed.`

**PASS:** roll / pitch / yaw on the next attitude message are all near zero
(within sensor noise, typically <0.1 deg). Gently rotating the device
afterward changes the values relative to the new zero, not the original
absolute frame.

**FAIL:** values unchanged from before the call, or service returns failure
on a connected device.

## Step 5 — `reset_acc_bias` (T3)

Goal: confirm the accelerometer compensation register is cleared and
persisted, and that a follow-up `reset_device` is required to apply it.

The handler holds an internal mutex (`service_acc_bias_mtx_`) so concurrent
calls serialize the flash write.

1. (Optional, for a clear before/after) read the current acceleration with
   the device level: `ros2 topic echo /vectornav/imu --once`.
2. Call:

```bash
ros2 service call /vectornav/reset_acc_bias std_srvs/srv/Trigger {}
```

**Expected response:** `success: True`, message
`Bias register set to zero. Please reset the device to apply.`

3. Apply the change:

```bash
ros2 service call /vectornav/reset_device std_srvs/srv/Trigger {}
```

4. After reconnection, read acceleration again.

**PASS:** the acceleration vector after the reset cycle reflects the cleared
bias (typically slightly different from before, by the previous bias value).

**FAIL:** service returns failure, or the bias is not actually persisted —
test by power-cycling the sensor and reading acceleration again; the values
should match step 4 (after reset), not step 1 (before).

**Concurrency check (optional):** spam parallel calls in a short window:

```bash
for i in 1 2 3 4 5; do
  ros2 service call /vectornav/reset_acc_bias std_srvs/srv/Trigger {} &
done
wait
```

**PASS:** all five complete without hang or SDK error. The mutex serializes
flash writes.

## Step 6 — Sensor disconnected behavior

Goal: confirm services degrade safely when the device is unplugged.

1. Unplug the sensor.
2. Wait until `reconnect_ms` is in its retry loop (visible in node log).
3. Call any of the three services.

**Expected response:**

```
success: False
message: "<exception text from the SDK or 'sensor handle is null'>"
```

**PASS:** all three services return `success: False`. The node does not
crash. After replugging, the next service call should succeed.

**FAIL:** node terminates from an uncaught SDK exception, or services return
`success: True` while no sensor is present (would indicate the try/catch
guard is missing or the null check is broken).

## Step 7 — Disable parameter

```bash
ros2 run vectornav vectornav --ros-args -p enable_diagnostic_services:=false
```

```bash
ros2 service list | grep vectornav
ros2 service call /vectornav/reset_device std_srvs/srv/Trigger {}
```

**PASS:**
- The three diagnostic services are absent from `ros2 service list`.
- The call returns `service not available`.
- Topics still publish normally — disabling the services does not affect
  the rest of the driver.

## Step 8 — Reconnect regression

Goal: confirm the existing `reconnect_ms` automatic reconnect was not
disturbed by the new code paths.

1. Start the node with `reconnect_ms` at its default (`500`).
2. Unplug the sensor mid-stream.
3. Wait, observe the log.
4. Replug the sensor.

**PASS:** within ≈1 second of replug, topics resume. No code paths from this
branch (the diagnostic services or their parameter) appear in the reconnect
log lines.

**FAIL:** reconnect no longer fires, fires at a wrong cadence, or
`enable_diagnostic_services` somehow gates reconnect logic.

## Composable-node parity

If you load the driver as a component (the original snslab use case), repeat
Steps 2, 3, 7 with the component loaded into a container. The services
should appear under the component's namespace and behave identically. This
exists because the patches were applied through the lifetime hook called
after `connect()`, which runs in both standalone and composable modes.

## What to report back

For each step, PASS / FAIL / N-A. For any FAIL, capture:

- The full service response.
- The relevant node log window (5 seconds before and after).
- Whether the device was actually present (`ls /dev/ttyUSB*`).

If everything passes, the branch is ready to merge to `humble`.
