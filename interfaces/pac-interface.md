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

1. **Standalone Device**: A dedicated polar alignment correction device (e.g., [Avalon Universal Polar Alignment System](https://www.avalon-instruments.com/products-menu/upas/universal-polar-alignment-system-detail) or the [MLAstro Robotic Polar Alignment](https://mlastro.com/)) that mechanically adjusts the mount's polar axis.

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
| `PAC_CAN_HOME` | Device supports homing — setting, returning to, and resetting a home position (`PAC_HOME` property) |
| `PAC_HAS_BACKLASH` | Device supports backlash compensation (`PAC_BACKLASH_ENABLED` / `PAC_BACKLASH_STEPS` properties) |
| `PAC_CAN_SYNC` | Device supports syncing the current position to a reference value (`PAC_SYNC` property) |

```cpp
MLAstroRPA::MLAstroRPA() : PACInterface(this)
{
    setVersion(1, 0);
    SetCapability(PAC_HAS_SPEED    |
                  PAC_CAN_REVERSE  |
                  PAC_HAS_POSITION |
                  PAC_CAN_HOME     |
                  PAC_HAS_BACKLASH |
                  PAC_CAN_SYNC);
}
```

### 3. Implement the Virtual Methods

The `INDI::PACInterface` base class defines virtual methods that you override to drive your hardware:

| Method | Required | Description |
|--------|----------|-------------|
| `MoveAZ(double degrees)` | **Yes** | Move the azimuth axis (+East, −West) |
| `MoveALT(double degrees)` | **Yes** | Move the altitude axis (+North, −South) |
| `MoveBoth(double azDegrees, double altDegrees)` | **Recommended** | Move both axes in a single coordinated operation (see [Coordinated Dual-Axis Movement](#coordinated-dual-axis-movement)) |
| `AbortMotion()` | **Yes** | Abort all in-progress axis motion |
| `SetPACSpeed(uint16_t speed)` | When `PAC_HAS_SPEED` | Set motor speed |
| `ReverseAZ(bool enabled)` | When `PAC_CAN_REVERSE` | Reverse azimuth axis direction |
| `ReverseALT(bool enabled)` | When `PAC_CAN_REVERSE` | Reverse altitude axis direction |
| `SetHome()` | When `PAC_CAN_HOME` | Mark the current position as the home position |
| `GoHome()` | When `PAC_CAN_HOME` | Return both axes to the home position |
| `ResetHome()` | When `PAC_CAN_HOME` | Clear the stored home position |
| `SyncAZ(double degrees)` | When `PAC_CAN_SYNC` | Sync the azimuth axis to the given value |
| `SyncALT(double degrees)` | When `PAC_CAN_SYNC` | Sync the altitude axis to the given value |
| `SetBacklashEnabled(bool enabled)` | When `PAC_HAS_BACKLASH` | Enable or disable backlash compensation |
| `SetBacklashAZ(int32_t steps)` | When `PAC_HAS_BACKLASH` | Set azimuth backlash in steps |
| `SetBacklashALT(int32_t steps)` | When `PAC_HAS_BACKLASH` | Set altitude backlash in steps |

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
| `MANUAL_AZ_STEP` | Azimuth step in degrees (+East / −West). Writing a non-zero value triggers axis movement. |
| `MANUAL_ALT_STEP` | Altitude step in degrees (+North / −South). Writing a non-zero value triggers axis movement. |

When **both** elements are non-zero in the same write, `processNumber()` calls `MoveBoth(azStep, altStep)` instead of the individual methods. If only one element is non-zero, `MoveAZ()` or `MoveALT()` is called directly. See [Coordinated Dual-Axis Movement](#coordinated-dual-axis-movement) for details.

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
| `PAC_HOME` | `PAC_CAN_HOME` | Switch | Set / Return to / Reset the home position |
| `PAC_SYNC` | `PAC_CAN_SYNC` | Number | Write desired AZ and ALT reference values to sync |
| `PAC_BACKLASH_ENABLED` | `PAC_HAS_BACKLASH` | Switch | Enable or disable backlash compensation |
| `PAC_BACKLASH_STEPS` | `PAC_HAS_BACKLASH` | Number | Backlash compensation in steps per axis |

#### PAC_POSITION elements (PAC_HAS_POSITION)

| Element | Description |
|---------|-------------|
| `POSITION_AZ` | Current azimuth position in degrees (−360 to +360) |
| `POSITION_ALT` | Current altitude position in degrees (−90 to +90) |

Drivers should update this property periodically (e.g. from `TimerHit()`).

#### PAC_HOME elements (PAC_CAN_HOME)

| Element | Description |
|---------|-------------|
| `HOME_SET` | Mark the current position as home — calls `SetHome()` |
| `HOME_GO` | Return to the stored home position — calls `GoHome()` |
| `HOME_RESET` | Clear the stored home position — calls `ResetHome()` |

The property state is set to `IPS_BUSY` while homing is in progress, `IPS_OK` on success, and `IPS_ALERT` on failure.

#### PAC_SYNC elements (PAC_CAN_SYNC)

| Element | Description |
|---------|-------------|
| `SYNC_AZ` | Azimuth reference value in degrees |
| `SYNC_ALT` | Altitude reference value in degrees |

Writing to this property calls `SyncAZ()` and `SyncALT()` with the provided values. Depending on the hardware, the device may zero both axes simultaneously (the most common behaviour) or accept arbitrary reference values.

#### PAC_BACKLASH_STEPS elements (PAC_HAS_BACKLASH)

| Element | Description |
|---------|-------------|
| `BACKLASH_AZ` | Azimuth backlash compensation in motor steps (0–10000) |
| `BACKLASH_ALT` | Altitude backlash compensation in motor steps (0–10000) |

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

### MoveBoth

```cpp
virtual IPState MoveBoth(double azDegrees, double altDegrees);
```

Move both axes in a single coordinated operation. Called by `processNumber()` when **both** `MANUAL_AZ_STEP` and `MANUAL_ALT_STEP` are non-zero in the same write.

The **default implementation** calls `MoveAZ(azDegrees)` followed by `MoveALT(altDegrees)` in sequence — fully backward-compatible for drivers that do not override it.

Drivers should override `MoveBoth` when their hardware supports a native dual-axis command:

| Hardware type | Recommended override |
|---|---|
| Native dual-axis command (e.g. GRBL `$J=G91G21X…Y… F…`) | Single command — both axes move simultaneously |
| Shared angle registers with internal sequencing (e.g. MLAstro RPA `AAll:1`) | Single chained command — device sequences AZ→ALT internally |

**Returns:**
- `IPS_OK` — all movement completed immediately
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

### SetHome

```cpp
virtual bool SetHome();
```

Mark the current position as the home position. Called when the user presses **Set Home** in the `PAC_HOME` property. Only called when `PAC_CAN_HOME` is set.

**Returns:** `true` if home was stored successfully, `false` otherwise.

### GoHome

```cpp
virtual IPState GoHome();
```

Command the device to return both axes to the previously stored home position. Called when the user presses **Return Home**. Only called when `PAC_CAN_HOME` is set.

**Returns:**
- `IPS_OK` — movement to home completed immediately
- `IPS_BUSY` — homing in progress (update `HomeSP` state when finished)
- `IPS_ALERT` — error (e.g. no home position set)

### ResetHome

```cpp
virtual bool ResetHome();
```

Clear the stored home position. Called when the user presses **Reset Home**. Only called when `PAC_CAN_HOME` is set.

**Returns:** `true` if the home position was cleared, `false` otherwise.

### SyncAZ

```cpp
virtual bool SyncAZ(double degrees);
```

Synchronise the azimuth axis to treat its current position as `degrees`. Many devices only support zeroing both axes simultaneously (`degrees` is ignored); others accept an arbitrary reference. Only called when `PAC_CAN_SYNC` is set.

**Returns:** `true` if successful, `false` otherwise.

### SyncALT

```cpp
virtual bool SyncALT(double degrees);
```

Synchronise the altitude axis to treat its current position as `degrees`. Only called when `PAC_CAN_SYNC` is set.

**Returns:** `true` if successful, `false` otherwise.

### SetBacklashEnabled

```cpp
virtual bool SetBacklashEnabled(bool enabled);
```

Enable or disable backlash compensation globally. Called when the user toggles `PAC_BACKLASH_ENABLED`. Only called when `PAC_HAS_BACKLASH` is set.

**Returns:** `true` if the setting was applied, `false` otherwise.

### SetBacklashAZ

```cpp
virtual bool SetBacklashAZ(int32_t steps);
```

Set the backlash compensation amount for the azimuth axis in motor steps. Only called when `PAC_HAS_BACKLASH` is set.

**Returns:** `true` if the value was applied, `false` otherwise.

### SetBacklashALT

```cpp
virtual bool SetBacklashALT(int32_t steps);
```

Set the backlash compensation amount for the altitude axis in motor steps. Only called when `PAC_HAS_BACKLASH` is set.

**Returns:** `true` if the value was applied, `false` otherwise.

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

        // PACInterface – single-axis movement (required)
        IPState MoveAZ(double degrees) override;
        IPState MoveALT(double degrees) override;

        // PACInterface – coordinated dual-axis movement (recommended)
        IPState MoveBoth(double azDegrees, double altDegrees) override;

        // PACInterface – abort, speed, and reverse
        bool AbortMotion() override;
        bool SetPACSpeed(uint16_t speed) override;   // PAC_HAS_SPEED
        bool ReverseAZ(bool enabled) override;       // PAC_CAN_REVERSE
        bool ReverseALT(bool enabled) override;      // PAC_CAN_REVERSE

        // PACInterface – home management
        bool    SetHome() override;                  // PAC_CAN_HOME
        IPState GoHome() override;                   // PAC_CAN_HOME
        bool    ResetHome() override;                // PAC_CAN_HOME

        // PACInterface – sync
        bool SyncAZ(double degrees) override;        // PAC_CAN_SYNC
        bool SyncALT(double degrees) override;       // PAC_CAN_SYNC

        // PACInterface – backlash
        bool SetBacklashEnabled(bool enabled) override; // PAC_HAS_BACKLASH
        bool SetBacklashAZ(int32_t steps) override;     // PAC_HAS_BACKLASH
        bool SetBacklashALT(int32_t steps) override;    // PAC_HAS_BACKLASH

    private:
        int  m_MovingAxes {0};
        bool m_IsHomed    {false};
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

    SetCapability(PAC_HAS_SPEED    |
                  PAC_CAN_REVERSE  |
                  PAC_HAS_POSITION |
                  PAC_CAN_HOME     |
                  PAC_HAS_BACKLASH |
                  PAC_CAN_SYNC);

    setDriverInterface(AUX_INTERFACE | PAC_INTERFACE);
}

// ---------------------------------------------------------------------------
// Properties
// ---------------------------------------------------------------------------

bool MyPACDriver::initProperties()
{
    INDI::DefaultDevice::initProperties();

    PACI::initProperties(MAIN_CONTROL_TAB);

    // Optionally adjust the speed range after PACI::initProperties():
    SpeedNP[0].setMin(1);
    SpeedNP[0].setMax(5);
    SpeedNP[0].setStep(1);
    SpeedNP[0].setValue(3);

    addAuxControls();
    return true;
}

bool MyPACDriver::updateProperties()
{
    INDI::DefaultDevice::updateProperties();
    PACI::updateProperties();
    return true;
}

bool MyPACDriver::saveConfigItems(FILE *fp)
{
    INDI::DefaultDevice::saveConfigItems(fp);
    // Saves SpeedNP, AZReverseSP, ALTReverseSP, HomeSP, SyncNP,
    // BacklashSP, BacklashNP (as enabled by capabilities).
    PACI::saveConfigItems(fp);
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
    LOG_INFO("Motion aborted.");
    m_MovingAxes = 0;
    return true;
}

// ---------------------------------------------------------------------------
// Speed and reverse
// ---------------------------------------------------------------------------

bool MyPACDriver::SetPACSpeed(uint16_t speed)
{
    LOGF_INFO("Speed set to %u.", speed);
    return true;
}

bool MyPACDriver::ReverseAZ(bool enabled)
{
    LOGF_INFO("Azimuth direction reverse %s.", enabled ? "enabled" : "disabled");
    return true;
}

bool MyPACDriver::ReverseALT(bool enabled)
{
    LOGF_INFO("Altitude direction reverse %s.", enabled ? "enabled" : "disabled");
    return true;
}

// ---------------------------------------------------------------------------
// Home management (PAC_CAN_HOME)
// ---------------------------------------------------------------------------

bool MyPACDriver::SetHome()
{
    // TODO: send SetHome command to hardware
    m_IsHomed = true;
    LOG_INFO("Home position stored.");
    return true;
}

IPState MyPACDriver::GoHome()
{
    if (!m_IsHomed)
    {
        LOG_ERROR("No home position set. Please set home first.");
        return IPS_ALERT;
    }
    // TODO: send GoHome command and start async tracking.
    // When complete, set HomeSP state to IPS_OK and call apply().
    LOG_INFO("Returning to home position...");
    return IPS_BUSY;
}

bool MyPACDriver::ResetHome()
{
    m_IsHomed = false;
    LOG_INFO("Home position cleared.");
    return true;
}

// ---------------------------------------------------------------------------
// Sync (PAC_CAN_SYNC)
// ---------------------------------------------------------------------------

bool MyPACDriver::SyncAZ(double degrees)
{
    LOGF_INFO("AZ synced to %.4f degrees.", degrees);
    return true;
}

bool MyPACDriver::SyncALT(double degrees)
{
    LOGF_INFO("ALT synced to %.4f degrees.", degrees);
    return true;
}

// ---------------------------------------------------------------------------
// Backlash (PAC_HAS_BACKLASH)
// ---------------------------------------------------------------------------

bool MyPACDriver::SetBacklashEnabled(bool enabled)
{
    LOGF_INFO("Backlash compensation %s.", enabled ? "enabled" : "disabled");
    return true;
}

bool MyPACDriver::SetBacklashAZ(int32_t steps)
{
    LOGF_INFO("AZ backlash set to %d steps.", steps);
    return true;
}

bool MyPACDriver::SetBacklashALT(int32_t steps)
{
    LOGF_INFO("ALT backlash set to %d steps.", steps);
    return true;
}

// ---------------------------------------------------------------------------
// Single-axis movement
// ---------------------------------------------------------------------------

IPState MyPACDriver::MoveAZ(double degrees)
{
    LOGF_INFO("Moving azimuth %.4f deg %s.", std::abs(degrees), degrees >= 0 ? "East" : "West");
    m_MovingAxes++;
    // TODO: command hardware; call completionCallback() when done.
    return IPS_BUSY;
}

IPState MyPACDriver::MoveALT(double degrees)
{
    LOGF_INFO("Moving altitude %.4f deg %s.", std::abs(degrees), degrees >= 0 ? "North" : "South");
    m_MovingAxes++;
    return IPS_BUSY;
}

// ---------------------------------------------------------------------------
// Coordinated dual-axis movement (recommended override)
// ---------------------------------------------------------------------------

IPState MyPACDriver::MoveBoth(double azDegrees, double altDegrees)
{
    // TODO: if the hardware has a single combined command, send it here.
    // Otherwise the default base-class implementation (sequential MoveAZ +
    // MoveALT) is used automatically and you can omit this override entirely.
    LOGF_INFO("MoveBoth: AZ %.4f deg, ALT %.4f deg.", azDegrees, altDegrees);
    m_MovingAxes += 2;
    // TODO: command hardware
    return IPS_BUSY;
}
```

### Step 3: Create the CMakeLists.txt Entry

If you are adding the driver inside the INDI source tree under `drivers/auxiliary/`:

```cmake
# Inside drivers/auxiliary/CMakeLists.txt
add_executable(indi_mypac mypacdriver.cpp)
target_link_libraries(indi_mypac indidriver)
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
            setDriverInterface(TELESCOPE_INTERFACE | PAC_INTERFACE);
            SetCapability(PAC_HAS_SPEED | PAC_CAN_REVERSE | PAC_CAN_HOME);
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
        IPState MoveAZ(double degrees) override;
        IPState MoveALT(double degrees) override;
        bool AbortMotion() override;
        bool SetPACSpeed(uint16_t speed) override;
        bool ReverseAZ(bool enabled) override;
        bool ReverseALT(bool enabled) override;
        bool SetHome() override;
        IPState GoHome() override;
        bool ResetHome() override;
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

### Completing an Asynchronous GoHome

When `GoHome()` returns `IPS_BUSY`, you must mark `HomeSP` as complete in your timer or poll loop:

```cpp
void MyPACDriver::TimerHit()
{
    if (!isConnected())
        return;

    // Poll hardware status
    bool stillHoming = queryHardwareIsHoming();

    if (!stillHoming && HomeSP.getState() == IPS_BUSY)
    {
        HomeSP.setState(IPS_OK);
        HomeSP.apply();
        LOG_INFO("Homing complete.");
    }

    SetTimer(getCurrentPollingPeriod());
}
```

## Reporting Position (PAC_HAS_POSITION)

If your hardware can report its current axis offset, set `PAC_HAS_POSITION` in `SetCapability()`. The `PAC_POSITION` property will then be defined automatically when the device connects.

Update the position values from your `TimerHit()` (or equivalent):

```cpp
void MyPACDriver::TimerHit()
{
    if (!isConnected())
        return;

    double azPos = 0, altPos = 0;
    if (getHardwarePosition(azPos, altPos))
    {
        PositionNP[POSITION_AZ].setValue(azPos);
        PositionNP[POSITION_ALT].setValue(altPos);
        PositionNP.setState(IPS_OK);
        PositionNP.apply();
    }

    SetTimer(getCurrentPollingPeriod());
}
```

## Coordinated Dual-Axis Movement

When a Polar Alignment Assistant needs to apply a correction in both azimuth and altitude at the same time, it sends a single `PAC_MANUAL_ADJUSTMENT` write with both elements non-zero. `processNumber()` detects this and calls `MoveBoth()` instead of the two individual methods.

### How `processNumber()` dispatches movement

```
PAC_MANUAL_ADJUSTMENT written
  ├─ MANUAL_AZ_STEP != 0 AND MANUAL_ALT_STEP != 0
  │      → MoveBoth(azStep, altStep)
  ├─ MANUAL_AZ_STEP != 0 only
  │      → MoveAZ(azStep)
  └─ MANUAL_ALT_STEP != 0 only
         → MoveALT(altStep)
```

### Default behaviour (no override)

If a driver does **not** override `MoveBoth`, the base-class implementation calls `MoveAZ()` and then `MoveALT()` in sequence. This is safe and correct for any driver that already implements single-axis movement.

### Overriding MoveBoth for simultaneous motion

Override `MoveBoth` when your hardware can move both axes in a single command:

**Example — Avalon UPAS (GRBL multi-axis jog):**
```cpp
IPState AvalonUPAS::MoveBoth(double azDegrees, double altDegrees)
{
    const double mmAZ    = azDegrees  * GearRatioNP[GEAR_AZ].getValue();
    const double mmALT   = altDegrees * GearRatioNP[GEAR_ALT].getValue();
    const double feedRate = SpeedNP[0].getValue();

    char cmd[DRIVER_LEN] = {0};
    snprintf(cmd, DRIVER_LEN, "$J=G91G21X%.4fY%.4f F%.0f", mmAZ, mmALT, feedRate);

    char res[DRIVER_LEN] = {0};
    if (!sendCommand(cmd, res) || strncmp(res, "ok", 2) != 0)
        return IPS_ALERT;

    m_IsMoving = true;
    return IPS_BUSY;
}
```

**Example — MLAstro RPA (chained angle registers + `AAll:1`):**
```cpp
IPState MLAstroRPA::MoveBoth(double azDegrees, double altDegrees)
{
    int azD, azM, azS; bool azPos;
    degreesToDMS(azDegrees,  azD, azM, azS, azPos);

    int altD, altM, altS; bool altPos;
    degreesToDMS(altDegrees, altD, altM, altS, altPos);

    char cmd[DRIVER_LEN] = {0};
    snprintf(cmd, DRIVER_LEN,
             "AzED:%d,AzEM:%d,AzES:%d,AzDi:%d,"
             "AlED:%d,AlEM:%d,AlES:%d,AlDi:%d,AAll:1",
             azD, azM, azS, azPos ? 1 : 0,
             altD, altM, altS, altPos ? 1 : 0);

    char res[DRIVER_LEN] = {0};
    if (!sendCommand(cmd, res) || strncmp(res, "ok", 2) != 0)
        return IPS_ALERT;

    m_IsMoving = true;
    return IPS_BUSY;
}
```

> **Note:** For the MLAstro RPA the `AAll:1` command causes the device to execute AZ first and then ALT internally. The driver does not need to sequence them — it only needs to detect completion via telemetry polling.

## Best Practices

When implementing the PAC interface, follow these best practices:

- **Implement simulation mode** — check `isSimulation()` and provide simulated motion with realistic durations so clients can be tested without hardware.
- **Set capabilities in the constructor** — call `SetCapability()` before `initProperties()` runs.
- **Adjust speed range after `PACI::initProperties()`** — the default range is 1–10; modify `SpeedNP[0]` min/max/step to match your hardware.
- **Override `MoveBoth` when your hardware has a native dual-axis command** — the default implementation (sequential `MoveAZ` + `MoveALT`) is safe, but a single combined command is faster and more accurate when supported.
- **Track moving axes** — maintain an axis counter so `ManualAdjustmentNP` is only set to `IPS_OK` after *both* requested axes have finished.
- **Guard GoHome against missing home** — check whether a home position has been stored before issuing the command and return `IPS_ALERT` with an informative message if not.
- **Forward `saveConfigItems`** — call `PACI::saveConfigItems(fp)` to persist speed, reverse, home, sync, and backlash settings across sessions.
- **Do not save device-side settings in `saveConfigItems`** — settings persisted on the device itself (e.g. in FRAM via a Save&Reboot command) should be read back from hardware via telemetry, not pushed from the INDI config file on reconnect.
- **Handle abort gracefully** — stop all hardware motion immediately and reset `m_MovingAxes` to 0.
- **Use 4-decimal-place precision** — degree values from PAA tools are typically ±0.001°.
- **Document sign conventions** in your driver's log messages and user manual.

## Reference Drivers

The following drivers in the INDI source tree implement the PAC interface and can serve as implementation references:

| Driver | Source file | Capabilities | `MoveBoth` override |
|--------|-------------|--------------|---------------------|
| PAC Simulator | `drivers/auxiliary/pac_simulator.cpp` | `PAC_HAS_SPEED \| PAC_CAN_REVERSE` | No (uses default) |
| Avalon UPAS | `drivers/auxiliary/avalon_upas.cpp` | `PAC_HAS_SPEED \| PAC_CAN_REVERSE \| PAC_HAS_POSITION` | Yes — single GRBL `$J=G91G21X…Y…` command |
| MLAstro RPA | `drivers/auxiliary/mlastro_rpa.cpp` | `PAC_HAS_SPEED \| PAC_CAN_REVERSE \| PAC_HAS_POSITION \| PAC_CAN_HOME \| PAC_HAS_BACKLASH \| PAC_CAN_SYNC` | Yes — chained `AAll:1` command |

## Conclusion

The PAC interface provides a standardised way for clients (such as the KStars Polar Alignment Assistant) to apply mechanical polar-alignment corrections. Whether implementing a standalone correction device or adding PAC capability to an existing mount driver, the interface requires only a small set of virtual methods and a minimal amount of property-forwarding boilerplate. The optional `PAC_CAN_HOME`, `PAC_HAS_BACKLASH`, and `PAC_CAN_SYNC` capabilities allow more advanced hardware — such as the MLAstro RPA — to expose their full feature set through a single, consistent interface.

For more information, refer to the [INDI Library Documentation](https://www.indilib.org/api/index.html) and the [INDI Driver Development Guide](https://www.indilib.org/develop/developer-manual/100-driver-development.html).
