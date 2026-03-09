---
title: PAC Interface
nav_order: 15
parent: Device Interfaces
---

# Implementing the PAC (Polar Alignment Correction) Interface

This guide provides a comprehensive overview of implementing the Polar Alignment Correction (PAC) interface in INDI drivers. It covers the basic structure of a PAC driver, how to implement the required methods, and how to handle device-specific functionality.

## Introduction to the PAC Interface

The PAC (Polar Alignment Correction) interface in INDI is designed for devices that provide automated polar alignment correction for equatorial mounts. Polar alignment is critical for astrophotography and long-exposure imaging, as misalignment causes star trailing and tracking errors.

The interface exposes a **manual step control** (`PAC_MANUAL_ADJUSTMENT`) that a client (e.g. Ekos Polar Alignment Assistant) uses to nudge the mount's polar axis by a signed number of degrees on either the azimuth or altitude axis.

The PAC interface can be implemented in two ways:

1. **Standalone Device**: A dedicated polar alignment correction device (e.g., [Avalon Universal Polar Alignment System](https://www.avalon-instruments.com/products-menu/upas/universal-polar-alignment-system-detail)) that mechanically adjusts the mount's polar axis.

2. **Integrated into a Mount Driver**: The PAC interface can be embedded directly into a telescope mount driver, allowing the mount to perform its own polar alignment corrections.

## Key Concepts for PAC Driver Development

Creating an INDI PAC driver involves four essential aspects:

### 1. Understand the Sign Convention

The PAC interface uses a specific sign convention for axis movements:

| Axis | Positive direction | Negative direction |
|------|-------------------|--------------------|
| Azimuth (`MANUAL_AZ_STEP`) | East | West |
| Altitude (`MANUAL_ALT_STEP`) | North (increase altitude) | South (decrease altitude) |

### 2. Set the Capability Flags

In your driver's constructor call `SetCapability()` with the appropriate bitmask to enable optional features:

| Flag | Description |
|------|-------------|
| `PAC_HAS_SPEED` | Device supports variable motor speed (`PAC_SPEED` property) |
| `PAC_CAN_REVERSE` | Device supports reversing each axis direction (`PAC_AZ_REVERSE` / `PAC_ALT_REVERSE` properties) |
| `PAC_HAS_POSITION` | Device can report its current axis position (`PAC_POSITION` property) |

```cpp
PACSimulator::PACSimulator() : PACInterface(this)
{
    setVersion(1, 0);
    SetCapability(PAC_HAS_SPEED | PAC_CAN_REVERSE);
}
```

### 3. Implement the Virtual Methods

The `INDI::PACInterface` base class defines virtual methods that you override to drive your hardware:

| Method | Required | Description |
|--------|----------|-------------|
| `MoveAZ(double degrees)` | **Yes** | Move the azimuth axis (+East, −West) |
| `MoveALT(double degrees)` | **Yes** | Move the altitude axis (+North, −South) |
| `AbortMotion()` | **Yes** | Abort all in-progress axis motion |
| `SetPACSpeed(uint16_t speed)` | When `PAC_HAS_SPEED` | Set motor speed |
| `ReverseAZ(bool enabled)` | When `PAC_CAN_REVERSE` | Reverse azimuth axis direction |
| `ReverseALT(bool enabled)` | When `PAC_CAN_REVERSE` | Reverse altitude axis direction |

### 4. Forward Property Processing

Your driver must forward property changes to the PAC interface and also save its config:

- Call `PACI::initProperties(group)` from `initProperties()`
- Call `PACI::updateProperties()` from `updateProperties()`
- Call `PACI::processSwitch()` from `ISNewSwitch()`
- Call `PACI::processNumber()` from `ISNewNumber()`
- Call `PACI::saveConfigItems(fp)` from `saveConfigItems()`

Note: `PACI` is a convenience alias for `INDI::PACInterface` defined in the header:
```cpp
using PACI = INDI::PACInterface;
```

## PAC Interface Properties

### Always-present properties

| Property name | Type | Permission | Description |
|---------------|------|------------|-------------|
| `PAC_MANUAL_ADJUSTMENT` | Number | Write-only | Signed azimuth and altitude step in degrees |
| `PAC_ABORT_MOTION` | Switch | Read/Write | Pressing Abort calls `AbortMotion()` |

#### PAC_MANUAL_ADJUSTMENT elements

| Element | Description |
|---------|-------------|
| `MANUAL_AZ_STEP` | Azimuth step in degrees (+East / −West). Writing a non-zero value immediately triggers `MoveAZ()`. |
| `MANUAL_ALT_STEP` | Altitude step in degrees (+North / −South). Writing a non-zero value immediately triggers `MoveALT()`. |

The property state reflects the overall motion status:
- `IPS_BUSY` — one or both axes still moving
- `IPS_ALERT` — one or both axes encountered an error
- `IPS_OK` — both axes completed successfully (or no movement was requested)

### Optional capability-gated properties

| Property name | Capability flag | Type | Description |
|---------------|----------------|------|-------------|
| `PAC_POSITION` | `PAC_HAS_POSITION` | Number (read-only) | Current azimuth and altitude offset in degrees |
| `PAC_SPEED` | `PAC_HAS_SPEED` | Number | Motor speed (default range 1–10) |
| `PAC_AZ_REVERSE` | `PAC_CAN_REVERSE` | Switch | Reverse azimuth axis direction |
| `PAC_ALT_REVERSE` | `PAC_CAN_REVERSE` | Switch | Reverse altitude axis direction |

#### PAC_POSITION elements (PAC_HAS_POSITION)

| Element | Description |
|---------|-------------|
| `POSITION_AZ` | Current azimuth position in degrees (−360 to +360) |
| `POSITION_ALT` | Current altitude position in degrees (−90 to +90) |

Drivers should update this property periodically (e.g. from `TimerHit()`).

## Virtual Methods Reference

### MoveAZ

```cpp
virtual IPState MoveAZ(double degrees);
```

Move the azimuth axis by the given number of degrees. Positive = East, negative = West.

The default implementation returns `IPS_ALERT`. Drivers must override this.

**Returns:**
- `IPS_OK` — movement completed immediately
- `IPS_BUSY` — movement in progress (update `ManualAdjustmentNP` state when done)
- `IPS_ALERT` — error occurred

### MoveALT

```cpp
virtual IPState MoveALT(double degrees);
```

Move the altitude axis by the given number of degrees. Positive = North (increase altitude), negative = South.

The default implementation returns `IPS_ALERT`. Drivers must override this.

**Returns:**
- `IPS_OK` — movement completed immediately
- `IPS_BUSY` — movement in progress (update `ManualAdjustmentNP` state when done)
- `IPS_ALERT` — error occurred

### AbortMotion

```cpp
virtual bool AbortMotion();
```

Abort all in-progress axis motion immediately. The default implementation logs an error and returns `false`. Drivers that support hardware abort must override this.

**Returns:** `true` if successfully aborted, `false` otherwise.

### SetPACSpeed

```cpp
virtual bool SetPACSpeed(uint16_t speed);
```

Set the motor speed. Only called when `PAC_HAS_SPEED` capability is set. The default implementation logs an error and returns `false`. Drivers with variable-speed hardware must override.

Speed range is defined by the driver. The default range is 1–10; drivers can adjust `SpeedNP[0]` min/max/step in `initProperties()` after calling `PACI::initProperties()`.

**Returns:** `true` if the speed was applied successfully, `false` otherwise.

### ReverseAZ

```cpp
virtual bool ReverseAZ(bool enabled);
```

Reverse (or restore) the azimuth axis movement direction. Only called when `PAC_CAN_REVERSE` is set.

**Returns:** `true` if successful, `false` otherwise.

### ReverseALT

```cpp
virtual bool ReverseALT(bool enabled);
```

Reverse (or restore) the altitude axis movement direction. Only called when `PAC_CAN_REVERSE` is set.

**Returns:** `true` if successful, `false` otherwise.

## Implementing a Standalone PAC Driver

The following example is based on the reference `PACSimulator` driver included with the INDI library.

### Step 1: Create the Header File

```cpp
// mypacdriver.h
#pragma once

#include "defaultdevice.h"
#include "indipacinterface.h"

class MyPACDriver : public INDI::DefaultDevice, public INDI::PACInterface
{
    public:
        MyPACDriver();
        virtual ~MyPACDriver() override = default;

        bool ISNewNumber(const char *dev, const char *name, double values[], char *names[], int n) override;
        bool ISNewSwitch(const char *dev, const char *name, ISState *states, char *names[], int n) override;

    protected:
        bool initProperties() override;
        bool updateProperties() override;
        bool saveConfigItems(FILE *fp) override;

        bool Connect() override;
        bool Disconnect() override;

        const char *getDefaultName() override
        {
            return "My PAC";
        }

        // PACInterface – single-axis movement
        // MoveAZ:  positive = East,  negative = West
        // MoveALT: positive = North, negative = South
        IPState MoveAZ(double degrees) override;
        IPState MoveALT(double degrees) override;

        // PACInterface – abort, speed, and reverse
        bool AbortMotion() override;
        bool SetPACSpeed(uint16_t speed) override;   // only needed with PAC_HAS_SPEED
        bool ReverseAZ(bool enabled) override;       // only needed with PAC_CAN_REVERSE
        bool ReverseALT(bool enabled) override;      // only needed with PAC_CAN_REVERSE

    private:
        // Track how many axes are still moving.
        int m_MovingAxes {0};
};
```

### Step 2: Create the Implementation File

```cpp
// mypacdriver.cpp
#include "mypacdriver.h"

#include <memory>

static std::unique_ptr<MyPACDriver> mypac(new MyPACDriver());

// ---------------------------------------------------------------------------
// Constructor / setup
// ---------------------------------------------------------------------------

MyPACDriver::MyPACDriver() : PACInterface(this)
{
    setVersion(1, 0);

    // Declare the capabilities your hardware supports.
    SetCapability(PAC_HAS_SPEED | PAC_CAN_REVERSE);

    // Include PAC_INTERFACE so clients (e.g. Ekos) discover this driver correctly.
    setDriverInterface(AUX_INTERFACE | PAC_INTERFACE);
}

// ---------------------------------------------------------------------------
// Properties
// ---------------------------------------------------------------------------

bool MyPACDriver::initProperties()
{
    INDI::DefaultDevice::initProperties();

    // Initialise PAC interface properties.
    PACI::initProperties(MAIN_CONTROL_TAB);

    // Optionally adjust the speed range after PACI::initProperties():
    SpeedNP[0].setMin(1);
    SpeedNP[0].setMax(5);
    SpeedNP[0].setStep(1);
    SpeedNP[0].setValue(1);

    addAuxControls();

    return true;
}

bool MyPACDriver::updateProperties()
{
    INDI::DefaultDevice::updateProperties();

    // PACI::updateProperties() defines / deletes PAC properties based on
    // connection state and the capability flags set in the constructor.
    PACI::updateProperties();

    return true;
}

// ---------------------------------------------------------------------------
// Property handlers
// ---------------------------------------------------------------------------

bool MyPACDriver::ISNewNumber(const char *dev, const char *name, double values[], char *names[], int n)
{
    if (dev != nullptr && strcmp(dev, getDeviceName()) == 0)
    {
        if (PACI::processNumber(dev, name, values, names, n))
            return true;
    }
    return INDI::DefaultDevice::ISNewNumber(dev, name, values, names, n);
}

bool MyPACDriver::ISNewSwitch(const char *dev, const char *name, ISState *states, char *names[], int n)
{
    if (dev != nullptr && strcmp(dev, getDeviceName()) == 0)
    {
        if (PACI::processSwitch(dev, name, states, names, n))
            return true;
    }
    return INDI::DefaultDevice::ISNewSwitch(dev, name, states, names, n);
}

bool MyPACDriver::saveConfigItems(FILE *fp)
{
    INDI::DefaultDevice::saveConfigItems(fp);
    // Saves SpeedNP, AZReverseSP, and ALTReverseSP (as enabled by capabilities).
    PACI::saveConfigItems(fp);
    return true;
}

// ---------------------------------------------------------------------------
// Connection
// ---------------------------------------------------------------------------

bool MyPACDriver::Connect()
{
    LOG_INFO("MyPAC connected.");
    return true;
}

bool MyPACDriver::Disconnect()
{
    LOG_INFO("MyPAC disconnected.");
    return true;
}

// ---------------------------------------------------------------------------
// Abort
// ---------------------------------------------------------------------------

bool MyPACDriver::AbortMotion()
{
    // TODO: send hardware abort command
    LOG_INFO("Alignment correction motion aborted.");
    m_MovingAxes = 0;
    return true;
}

// ---------------------------------------------------------------------------
// Speed and reverse (PAC_HAS_SPEED / PAC_CAN_REVERSE)
// ---------------------------------------------------------------------------

bool MyPACDriver::SetPACSpeed(uint16_t speed)
{
    // TODO: send speed command to hardware
    LOGF_INFO("Speed set to %u.", speed);
    return true;
}

bool MyPACDriver::ReverseAZ(bool enabled)
{
    // TODO: send reverse command to hardware
    LOGF_INFO("Azimuth direction reverse %s.", enabled ? "enabled" : "disabled");
    return true;
}

bool MyPACDriver::ReverseALT(bool enabled)
{
    // TODO: send reverse command to hardware
    LOGF_INFO("Altitude direction reverse %s.", enabled ? "enabled" : "disabled");
    return true;
}

// ---------------------------------------------------------------------------
// Single-axis movement
// ---------------------------------------------------------------------------

IPState MyPACDriver::MoveAZ(double degrees)
{
    const char *direction = (degrees >= 0) ? "East" : "West";
    LOGF_INFO("Moving azimuth: %.4f deg %s.", std::abs(degrees), direction);

    // TODO: send move command to hardware and start async tracking.
    m_MovingAxes++;

    // When the move completes (e.g. in a timer callback or hardware interrupt):
    //   m_MovingAxes--;
    //   if (m_MovingAxes <= 0) {
    //       m_MovingAxes = 0;
    //       ManualAdjustmentNP.setState(IPS_OK);
    //       ManualAdjustmentNP.apply();
    //   }

    return IPS_BUSY;
}

IPState MyPACDriver::MoveALT(double degrees)
{
    const char *direction = (degrees >= 0) ? "North" : "South";
    LOGF_INFO("Moving altitude: %.4f deg %s.", std::abs(degrees), direction);

    // TODO: send move command to hardware and start async tracking.
    m_MovingAxes++;

    return IPS_BUSY;
}
```

### Step 3: Create the CMakeLists.txt Entry

If you are adding the driver inside the INDI source tree under `drivers/auxiliary/`:

```cmake
# Inside drivers/auxiliary/CMakeLists.txt
add_executable(indi_mypac mypacdriver.cpp)
target_link_libraries(indi_mypac indibase)
install(TARGETS indi_mypac RUNTIME DESTINATION bin)
```

For an out-of-tree driver, use `find_package(INDI REQUIRED)` and link against `${INDI_LIBRARIES}`.

### Step 4: Create the XML Driver Description

Create `indi_mypac.xml`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<driversList>
   <devGroup group="Auxiliary">
      <device label="My PAC" manufacturer="YourCompany">
         <driver name="My PAC">indi_mypac</driver>
         <version>1.0</version>
      </device>
   </devGroup>
</driversList>
```

### Step 5: Build and Install

```bash
mkdir build && cd build
cmake ..
make
sudo make install
```

## Integrating PAC into a Mount Driver

To add PAC capabilities to an existing telescope mount driver, use multiple inheritance and include `PAC_INTERFACE` in `setDriverInterface()`:

```cpp
#include "inditelescope.h"
#include "indipacinterface.h"

class MyMountWithPAC : public INDI::Telescope, public INDI::PACInterface
{
    public:
        MyMountWithPAC() : PACInterface(this)
        {
            // Include PAC_INTERFACE in addition to TELESCOPE_INTERFACE.
            setDriverInterface(TELESCOPE_INTERFACE | PAC_INTERFACE);
            SetCapability(PAC_HAS_SPEED | PAC_CAN_REVERSE);
        }

        bool initProperties() override
        {
            INDI::Telescope::initProperties();
            PACI::initProperties(MAIN_CONTROL_TAB);
            addAuxControls();
            return true;
        }

        bool updateProperties() override
        {
            INDI::Telescope::updateProperties();
            PACI::updateProperties();
            return true;
        }

        bool ISNewNumber(const char *dev, const char *name, double values[], char *names[], int n) override
        {
            if (dev != nullptr && strcmp(dev, getDeviceName()) == 0)
            {
                if (PACI::processNumber(dev, name, values, names, n))
                    return true;
            }
            return INDI::Telescope::ISNewNumber(dev, name, values, names, n);
        }

        bool ISNewSwitch(const char *dev, const char *name, ISState *states, char *names[], int n) override
        {
            if (dev != nullptr && strcmp(dev, getDeviceName()) == 0)
            {
                if (PACI::processSwitch(dev, name, states, names, n))
                    return true;
            }
            return INDI::Telescope::ISNewSwitch(dev, name, states, names, n);
        }

        bool saveConfigItems(FILE *fp) override
        {
            INDI::Telescope::saveConfigItems(fp);
            PACI::saveConfigItems(fp);
            return true;
        }

    protected:
        // Implement MoveAZ / MoveALT using the mount's existing adjustment motors.
        IPState MoveAZ(double degrees) override;
        IPState MoveALT(double degrees) override;
        bool AbortMotion() override;
        bool SetPACSpeed(uint16_t speed) override;
        bool ReverseAZ(bool enabled) override;
        bool ReverseALT(bool enabled) override;
};
```

## Asynchronous Movement and State Reporting

When `MoveAZ()` or `MoveALT()` returns `IPS_BUSY`, the driver is responsible for updating `ManualAdjustmentNP` once motion finishes. A typical pattern using `INDI::Timer::singleShot()`:

```cpp
IPState MyPACDriver::MoveAZ(double degrees)
{
    const double duration = /* compute from speed */ 2.0;

    m_MovingAxes++;

    INDI::Timer::singleShot(static_cast<int>(duration * 1000), [this, degrees]()
    {
        LOGF_INFO("Azimuth move complete: %.4f deg.", std::abs(degrees));

        m_MovingAxes--;
        if (m_MovingAxes <= 0)
        {
            m_MovingAxes = 0;
            ManualAdjustmentNP.setState(IPS_OK);
            ManualAdjustmentNP.apply();
        }
    });

    return IPS_BUSY;
}
```

When both axes complete, set `ManualAdjustmentNP` to `IPS_OK` and call `apply()` so that snooping drivers (e.g. the telescope simulator) can read the applied correction.

## Reporting Position (PAC_HAS_POSITION)

If your hardware can report its current axis offset, set `PAC_HAS_POSITION` in `SetCapability()`. The `PAC_POSITION` property will then be defined automatically when the device connects.

Update the position values from your `TimerHit()` (or equivalent):

```cpp
void MyPACDriver::TimerHit()
{
    if (!isConnected())
        return;

    // Query hardware for current position and update.
    double azPos = 0, altPos = 0;
    if (getHardwarePosition(azPos, altPos))
    {
        PositionNP[POSITION_AZ].setValue(azPos);
        PositionNP[POSITION_ALT].setValue(altPos);
        PositionNP.setState(IPS_OK);
        PositionNP.apply();
    }

    SetTimer(POLLMS);
}
```

## Best Practices

When implementing the PAC interface, follow these best practices:

- **Implement simulation mode** — check `isSimulation()` and provide simulated motion with realistic durations so clients can be tested without hardware.
- **Set capabilities in the constructor** — call `SetCapability()` before `initProperties()` runs.
- **Adjust speed range after `PACI::initProperties()`** — the default range is 1–10; modify `SpeedNP[0]` min/max/step to match your hardware.
- **Track moving axes** — maintain an axis counter so `ManualAdjustmentNP` is only set to `IPS_OK` after *both* requested axes have finished.
- **Forward `saveConfigItems`** — call `PACI::saveConfigItems(fp)` to persist speed and reverse settings across sessions.
- **Handle abort gracefully** — stop all hardware motion immediately and reset `m_MovingAxes` to 0.
- **Use 4-decimal-place precision** — degree values from PAA tools are typically ±0.001°.
- **Document sign conventions** in your driver's log messages and user manual.

## Conclusion

The PAC interface provides a standardised way for clients (such as the KStars Polar Alignment Assistant) to apply mechanical polar-alignment corrections. Whether implementing a standalone correction device or adding PAC capability to an existing mount driver, the interface requires only six virtual methods and a small amount of property-forwarding boilerplate.

For more information, refer to the [INDI Library Documentation](https://www.indilib.org/api/index.html) and the [INDI Driver Development Guide](https://www.indilib.org/develop/developer-manual/100-driver-development.html).
