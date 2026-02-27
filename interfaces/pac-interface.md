---
title: PAC Interface
nav_order: 15
parent: Device Interfaces
---

# Implementing the PAC (Polar Alignment Correction) Interface

This guide provides a comprehensive overview of implementing the Polar Alignment Correction (PAC) interface in INDI drivers. It covers the basic structure of a PAC driver, how to implement the required methods, and how to handle device-specific functionality.

## Introduction to the PAC Interface

The PAC (Polar Alignment Correction) interface in INDI is designed for devices that provide automated polar alignment correction for equatorial mounts. Polar alignment is critical for astrophotography and long-exposure imaging, as misalignment causes star trailing and tracking errors.

The PAC interface can be implemented in two ways:

1. **Standalone Device**: A dedicated polar alignment correction device (e.g., [Avalon Universal Polar Alignment System](https://www.avalon-instruments.com/products-menu/upas/universal-polar-alignment-system-detail) that mechanically adjusts the mount's polar axis.

2. **Integrated into a Mount Driver**: The PAC interface can be embedded directly into a telescope mount driver, allowing the mount to perform its own polar alignment corrections.

The PAC interface works by accepting azimuth and altitude error values (typically from a polar alignment assistant tool like KStars PAA) and applying the necessary mechanical corrections to bring the mount's polar axis into alignment with the celestial pole.

## Key Concepts for PAC Driver Development

Creating an INDI PAC driver involves four essential aspects:

### 1. Understand the Sign Convention

The PAC interface uses a specific sign convention for errors and corrections:

**Error Values:**
- **Positive azimuth error**: Polar axis is displaced to the East
- **Negative azimuth error**: Polar axis is displaced to the West
- **Positive altitude error**: Polar axis is too high (above the celestial pole)
- **Negative altitude error**: Polar axis is too low (below the celestial pole)

**Correction Movements:**
- **Azimuth (MoveAZ)**: Positive value moves East, negative value moves West
- **Altitude (MoveALT)**: Positive value moves North (increases altitude), negative value moves South (decreases altitude)

The relationship between errors and corrections is inverted:
- A positive azimuth error (East) requires a negative correction (West): `MoveAZ(-azError)`
- A positive altitude error (too high) requires a negative correction (South): `MoveALT(-altError)`

### 2. Implement Virtual Functions

The `INDI::PACInterface` base class defines virtual methods that you must implement:

- **`StartCorrection(double azError, double altError)`**: Start automated correction using the provided error values
- **`AbortCorrection()`**: Abort any ongoing correction
- **`MoveAZ(double degrees)`**: Move the azimuth axis by the specified degrees
- **`MoveALT(double degrees)`**: Move the altitude axis by the specified degrees

### 3. Handle Property Processing

Your driver must forward property changes to the PAC interface:

- Call `PACInterface::processSwitch()` from your `ISNewSwitch()` method
- Call `PACInterface::processNumber()` from your `ISNewNumber()` method

### 4. Report Progress via Status Property

During correction operations, update the `CorrectionStatusLP` property to reflect the current state:
- **IPS_IDLE**: No correction in progress
- **IPS_BUSY**: Correction in progress
- **IPS_OK**: Correction completed successfully
- **IPS_ALERT**: Correction failed

## Prerequisites

Before implementing the PAC interface, you should have:

- Basic knowledge of C++ programming
- Understanding of the INDI protocol and architecture
- Familiarity with polar alignment concepts and procedures
- Development environment set up (compiler, build tools, etc.)
- INDI library installed

## PAC Interface Structure

The PAC interface consists of several key components:

### Base Class

`INDI::PACInterface` is a mixin class that can be combined with `INDI::DefaultDevice` (for standalone devices) or `INDI::Telescope` (for mount-integrated implementations) through multiple inheritance.

### Standard Properties

The PAC interface defines four standard properties:

| Property | Type | Description |
|----------|------|-------------|
| **ALIGNMENT_CORRECTION_ERROR** | Number | Azimuth and altitude error values in degrees |
| **ALIGNMENT_CORRECTION** | Switch | Start and Abort correction commands |
| **ALIGNMENT_CORRECTION_STATUS** | Light | Current status of the correction operation |
| **PAC_MANUAL_ADJUSTMENT** | Number | Manual azimuth and altitude step adjustments |

#### ALIGNMENT_CORRECTION_ERROR

This number property contains two elements:

- **AZ_ERROR**: Azimuth error in degrees (-10 to +10)
- **ALT_ERROR**: Altitude error in degrees (-10 to +10)

Clients (like KStars Polar Alignment Assistant) set these values based on their measurements.

#### ALIGNMENT_CORRECTION

This switch property contains two elements:

- **CORRECT**: Start the automated correction
- **ABORT**: Abort any ongoing correction

#### ALIGNMENT_CORRECTION_STATUS

This light property provides visual feedback on the correction status:

- **STATUS**: Shows IPS_BUSY during correction, IPS_OK when complete, IPS_ALERT on error

#### PAC_MANUAL_ADJUSTMENT

This number property allows manual fine-tuning:

- **MANUAL_AZ_STEP**: Azimuth step in degrees (+East/-West)
- **MANUAL_ALT_STEP**: Altitude step in degrees (+North/-South)

### Virtual Methods

The PAC interface defines four virtual methods:

#### StartCorrection

```cpp
virtual IPState StartCorrection(double azError, double altError);
```

Called when the client requests an automated correction. The default implementation calls `MoveAZ(-azError)` and `MoveALT(-altError)`. Override this method if your hardware supports a combined correction command.

**Returns:**
- `IPS_OK`: Correction completed immediately
- `IPS_BUSY`: Correction in progress (update `CorrectionStatusLP` when done)
- `IPS_ALERT`: Error occurred

#### AbortCorrection

```cpp
virtual IPState AbortCorrection();
```

Called when the client requests to abort a correction. Must be implemented by the driver.

**Returns:**
- `IPS_OK`: Successfully aborted
- `IPS_ALERT`: Error occurred

#### MoveAZ

```cpp
virtual IPState MoveAZ(double degrees);
```

Move the azimuth axis. Positive values move East, negative values move West.

**Returns:**
- `IPS_OK`: Movement completed immediately
- `IPS_BUSY`: Movement in progress
- `IPS_ALERT`: Error occurred (default implementation)

#### MoveALT

```cpp
virtual IPState MoveALT(double degrees);
```

Move the altitude axis. Positive values move North (increase altitude), negative values move South (decrease altitude).

**Returns:**
- `IPS_OK`: Movement completed immediately
- `IPS_BUSY`: Movement in progress
- `IPS_ALERT`: Error occurred (default implementation)

## Implementing a Standalone PAC Driver

Let's create a standalone PAC driver for a hypothetical polar alignment correction device called "MyPAC". This device has motors for both azimuth and altitude adjustment and communicates via serial/USB.

### Step 1: Create the Header File

Create a file named `mypacdriver.h` with the following content:

```cpp
#pragma once

#include <defaultdevice.h>
#include <indipacinterface.h>

class MyPACDriver : public INDI::DefaultDevice, public INDI::PACInterface
{
public:
    MyPACDriver();
    virtual ~MyPACDriver() = default;

    // DefaultDevice overrides
    virtual const char *getDefaultName() override;
    virtual bool initProperties() override;
    virtual bool updateProperties() override;
    virtual bool ISNewNumber(const char *dev, const char *name, double values[], char *names[], int n) override;
    virtual bool ISNewSwitch(const char *dev, const char *name, ISState *states, char *names[], int n) override;

    // PACInterface overrides
    virtual IPState StartCorrection(double azError, double altError) override;
    virtual IPState AbortCorrection() override;
    virtual IPState MoveAZ(double degrees) override;
    virtual IPState MoveALT(double degrees) override;

protected:
    // Connection overrides
    virtual bool Connect() override;
    virtual bool Disconnect() override;

    // Periodic updates
    virtual void TimerHit() override;

    // Helpers
    bool sendCommand(const char *cmd, char *res = nullptr, int reslen = 0);
    bool isMoving();

private:
    // Device handle
    int PortFD = -1;

    // Movement state
    bool AzMoving = false;
    bool AltMoving = false;

    // Custom properties
    INDI::PropertyNumber MovementSpeedNP {1};
};
```

### Step 2: Create the Implementation File

Create a file named `mypacdriver.cpp` with the following content:

```cpp
#include "mypacdriver.h"

#include <memory>
#include <string.h>
#include <unistd.h>
#include <connectionplugins/connectionserial.h>

// We declare an auto pointer to MyPACDriver
static std::unique_ptr<MyPACDriver> mypac(new MyPACDriver());

MyPACDriver::MyPACDriver()
    : PACInterface(this)
{
    setVersion(1, 0);

    // Set the driver interface to PAC_INTERFACE
    setDriverInterface(PAC_INTERFACE);
}

const char *MyPACDriver::getDefaultName()
{
    return "My PAC";
}

bool MyPACDriver::initProperties()
{
    // Initialize the parent's properties
    INDI::DefaultDevice::initProperties();

    // Initialize PAC interface properties
    PACInterface::initProperties(MAIN_CONTROL_TAB);

    // Add custom properties
    MovementSpeedNP[0].fill("SPEED", "Speed (deg/s)", "%.2f", 0.1, 5.0, 0.1, 1.0);
    MovementSpeedNP.fill(getDeviceName(), "MOVEMENT_SPEED", "Movement Speed",
                         MAIN_CONTROL_TAB, IP_RW, 0, IPS_IDLE);

    // Add debug, simulation, and configuration controls
    addAuxControls();

    return true;
}

bool MyPACDriver::updateProperties()
{
    // Call the parent's updateProperties
    INDI::DefaultDevice::updateProperties();

    // Call PAC interface updateProperties
    PACInterface::updateProperties();

    if (isConnected())
    {
        defineProperty(MovementSpeedNP);
    }
    else
    {
        deleteProperty(MovementSpeedNP);
    }

    return true;
}

bool MyPACDriver::ISNewNumber(const char *dev, const char *name, double values[], char *names[], int n)
{
    // Check if the message is for this device
    if (dev != nullptr && strcmp(dev, getDeviceName()) == 0)
    {
        // Handle custom properties
        if (MovementSpeedNP.isNameMatch(name))
        {
            MovementSpeedNP.update(values, names, n);
            MovementSpeedNP.setState(IPS_OK);
            MovementSpeedNP.apply();
            return true;
        }

        // Let PAC interface handle its properties
        if (PACInterface::processNumber(dev, name, values, names, n))
            return true;
    }

    return INDI::DefaultDevice::ISNewNumber(dev, name, values, names, n);
}

bool MyPACDriver::ISNewSwitch(const char *dev, const char *name, ISState *states, char *names[], int n)
{
    // Check if the message is for this device
    if (dev != nullptr && strcmp(dev, getDeviceName()) == 0)
    {
        // Let PAC interface handle its properties
        if (PACInterface::processSwitch(dev, name, states, names, n))
            return true;
    }

    return INDI::DefaultDevice::ISNewSwitch(dev, name, states, names, n);
}

bool MyPACDriver::Connect()
{
    bool result = INDI::DefaultDevice::Connect();

    if (result)
    {
        // Get the file descriptor for the serial port
        PortFD = serialConnection->getPortFD();

        // Send a test command to verify the connection
        if (!sendCommand("PING\r\n"))
        {
            LOG_ERROR("Failed to communicate with the PAC device");
            return false;
        }

        // Start the timer for periodic updates
        SetTimer(POLLMS);

        LOG_INFO("PAC device connected successfully");
    }

    return result;
}

bool MyPACDriver::Disconnect()
{
    // Close the serial port
    if (PortFD > 0)
    {
        close(PortFD);
        PortFD = -1;
    }

    return INDI::DefaultDevice::Disconnect();
}

void MyPACDriver::TimerHit()
{
    if (!isConnected())
        return;

    // Check if movement is complete
    if (AzMoving || AltMoving)
    {
        if (!isMoving())
        {
            AzMoving = false;
            AltMoving = false;

            // Update status
            ManualAdjustmentNP.setState(IPS_OK);
            ManualAdjustmentNP.apply();

            // If this was part of an automated correction, check if complete
            if (CorrectionSP.getState() == IPS_BUSY)
            {
                CorrectionSP.setState(IPS_OK);
                CorrectionSP.reset();
                CorrectionSP.apply();
                CorrectionStatusLP[0].setState(IPS_OK);
                CorrectionStatusLP.apply();
                LOG_INFO("Alignment correction completed successfully.");
            }
        }
    }

    SetTimer(POLLMS);
}

IPState MyPACDriver::StartCorrection(double azError, double altError)
{
    if (CorrectionSP.getState() == IPS_BUSY)
    {
        LOG_WARN("Alignment correction is already in progress.");
        return IPS_BUSY;
    }

    LOGF_INFO("Starting alignment correction: AZ=%.4f deg, ALT=%.4f deg", azError, altError);

    // Apply corrections (inverted from error)
    const IPState azState = MoveAZ(-azError);
    const IPState altState = MoveALT(-altError);

    if (azState == IPS_ALERT || altState == IPS_ALERT)
        return IPS_ALERT;
    if (azState == IPS_BUSY || altState == IPS_BUSY)
        return IPS_BUSY;
    return IPS_OK;
}

IPState MyPACDriver::AbortCorrection()
{
    // Send abort command to the device
    if (!sendCommand("ABORT\r\n"))
    {
        LOG_ERROR("Failed to abort correction");
        return IPS_ALERT;
    }

    AzMoving = false;
    AltMoving = false;

    LOG_INFO("Alignment correction aborted.");
    return IPS_OK;
}

IPState MyPACDriver::MoveAZ(double degrees)
{
    const char *direction = (degrees >= 0) ? "EAST" : "WEST";

    LOGF_INFO("Moving azimuth: %.4f deg %s", std::abs(degrees), direction);

    // Send move command
    char cmd[32];
    snprintf(cmd, sizeof(cmd), "MOVE_AZ %s %.4f\r\n", direction, std::abs(degrees));

    if (!sendCommand(cmd))
    {
        LOG_ERROR("Failed to move azimuth axis");
        return IPS_ALERT;
    }

    AzMoving = true;
    return IPS_BUSY;
}

IPState MyPACDriver::MoveALT(double degrees)
{
    const char *direction = (degrees >= 0) ? "NORTH" : "SOUTH";

    LOGF_INFO("Moving altitude: %.4f deg %s", std::abs(degrees), direction);

    // Send move command
    char cmd[32];
    snprintf(cmd, sizeof(cmd), "MOVE_ALT %s %.4f\r\n", direction, std::abs(degrees));

    if (!sendCommand(cmd))
    {
        LOG_ERROR("Failed to move altitude axis");
        return IPS_ALERT;
    }

    AltMoving = true;
    return IPS_BUSY;
}

bool MyPACDriver::sendCommand(const char *cmd, char *res, int reslen)
{
    if (PortFD < 0)
    {
        LOG_ERROR("Serial port not open");
        return false;
    }

    // Write the command
    int nbytes_written = write(PortFD, cmd, strlen(cmd));
    if (nbytes_written < 0)
    {
        LOGF_ERROR("Error writing to PAC device: %s", strerror(errno));
        return false;
    }

    // If no response is expected, return success
    if (res == nullptr || reslen <= 0)
        return true;

    // Read the response
    int nbytes_read = read(PortFD, res, reslen - 1);
    if (nbytes_read < 0)
    {
        LOGF_ERROR("Error reading from PAC device: %s", strerror(errno));
        return false;
    }

    res[nbytes_read] = '\0';
    return true;
}

bool MyPACDriver::isMoving()
{
    char res[16];
    if (!sendCommand("STATUS\r\n", res, sizeof(res)))
        return false;

    int moving = 0;
    if (sscanf(res, "MOVING %d", &moving) != 1)
        return false;

    return moving != 0;
}
```

### Step 3: Create the CMakeLists.txt File

```cmake
cmake_minimum_required(VERSION 3.0)
project(indi-mypac CXX C)

include(GNUInstallDirs)
list(APPEND CMAKE_MODULE_PATH "${CMAKE_CURRENT_SOURCE_DIR}/cmake_modules/")

find_package(INDI REQUIRED)

include_directories(${CMAKE_CURRENT_BINARY_DIR})
include_directories(${CMAKE_CURRENT_SOURCE_DIR})
include_directories(${INDI_INCLUDE_DIR})

add_executable(indi_mypac mypacdriver.cpp)

target_link_libraries(indi_mypac ${INDI_LIBRARIES})

install(TARGETS indi_mypac RUNTIME DESTINATION bin)
```

### Step 4: Create the XML File

Create a file named `indi_mypac.xml`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<driversList>
   <devGroup group="Auxiliary">
      <device label="My PAC" manufacturer="INDI">
         <driver name="My PAC">indi_mypac</driver>
         <version>1.0</version>
      </device>
   </devGroup>
</driversList>
```

### Step 5: Build and Install

```bash
mkdir build
cd build
cmake ..
make
sudo make install
```

## Integrating PAC into a Mount Driver

To add PAC capabilities to an existing telescope mount driver, use multiple inheritance:

```cpp
#include <inditelescope.h>
#include <indipacinterface.h>

class MyMountWithPAC : public INDI::Telescope, public INDI::PACInterface
{
public:
    MyMountWithPAC();

    // Override initProperties to include PAC properties
    virtual bool initProperties() override;
    virtual bool updateProperties() override;

    // Forward to PAC interface
    virtual bool ISNewNumber(const char *dev, const char *name, double values[], char *names[], int n) override;
    virtual bool ISNewSwitch(const char *dev, const char *name, ISState *states, char *names[], int n) override;

    // Implement PAC virtual methods
    virtual IPState StartCorrection(double azError, double altError) override;
    virtual IPState AbortCorrection() override;
    virtual IPState MoveAZ(double degrees) override;
    virtual IPState MoveALT(double degrees) override;

protected:
    // ... other telescope methods
};

MyMountWithPAC::MyMountWithPAC()
    : PACInterface(this)
{
    // Include PAC_INTERFACE in addition to TELESCOPE_INTERFACE
    setDriverInterface(TELESCOPE_INTERFACE | PAC_INTERFACE);
}

bool MyMountWithPAC::initProperties()
{
    INDI::Telescope::initProperties();
    PACInterface::initProperties(MAIN_CONTROL_TAB);
    addAuxControls();
    return true;
}

bool MyMountWithPAC::updateProperties()
{
    INDI::Telescope::updateProperties();
    PACInterface::updateProperties();
    return true;
}

bool MyMountWithPAC::ISNewNumber(const char *dev, const char *name, double values[], char *names[], int n)
{
    if (dev != nullptr && strcmp(dev, getDeviceName()) == 0)
    {
        if (PACInterface::processNumber(dev, name, values, names, n))
            return true;
    }
    return INDI::Telescope::ISNewNumber(dev, name, values, names, n);
}

bool MyMountWithPAC::ISNewSwitch(const char *dev, const char *name, ISState *states, char *names[], int n)
{
    if (dev != nullptr && strcmp(dev, getDeviceName()) == 0)
    {
        if (PACInterface::processSwitch(dev, name, states, names, n))
            return true;
    }
    return INDI::Telescope::ISNewSwitch(dev, name, states, names, n);
}

// Implement MoveAZ and MoveALT to use the mount's existing motors
IPState MyMountWithPAC::MoveAZ(double degrees)
{
    // Use the mount's azimuth adjustment motor
    // This depends on your mount's specific hardware
    // ...
}

IPState MyMountWithPAC::MoveALT(double degrees)
{
    // Use the mount's altitude adjustment motor
    // ...
}
```

## Best Practices

When implementing the PAC interface, follow these best practices:

- **Implement simulation mode** to allow testing without hardware. Check `isSimulation()` and provide simulated responses.
- **Provide informative error messages** to help users troubleshoot issues.
- **Handle abort gracefully** - ensure the device stops moving immediately when abort is requested.
- **Update status promptly** - keep `CorrectionStatusLP` updated so clients can monitor progress.
- **Use appropriate precision** for error values - typically 4 decimal places for degree values.
- **Validate input ranges** - ensure error values are within reasonable limits before applying corrections.
- **Document sign conventions** clearly in your driver's documentation.
- **Consider safety limits** - prevent movements that could damage equipment or cause collisions.

## Conclusion

Implementing the PAC interface in INDI drivers enables automated polar alignment correction, significantly improving the user experience for astrophotographers. Whether implementing a standalone correction device or integrating PAC capabilities into a mount driver, the interface provides a standardized way for clients to measure and correct polar alignment errors.

For more information, refer to the [INDI Library Documentation](https://www.indilib.org/api/index.html) and the [INDI Driver Development Guide](https://www.indilib.org/develop/developer-manual/100-driver-development.html).