# One of two outputs always switched on - weekly planned

### Supported Devices:

(generally all LUA enabled NETIO devices)

- PowerPDU 4C
- NETIO 4C
- NETIO 4
- NETIO 4All

### Features:

- only one of two selected outputs is switched ON at one time
- if second output is switched on and the first output state is ON, the script ensures that the first output is switched OFF at first and after that the second output is switched on
- one day in week can be set up for automatic change states of selected outputs (useful for load balancing)


### Use case:

Two identical backup servers are changing online once a week according to changeDay, timeOFF, timeON and selected outputs. It was used for server load balancing.

### Configuration variables:

- **output1**
  - Number of the first output
  - Number from 1 to 4
- **output2**
  - Number of the second output
  - Number from 1 to 4

- **changeDay**
  - which day of week states of outputs should be changed
  - Number from 1 to 7 (1 - Monday, 7 - Sunday)
- **timeOFF**
  - Time when output which is ON should be turned OFF
  - String in format: "hh:mm" - hours:minutes
  - Hours are in 00-23 format
  - Example: "02:30"
- **timeON**
  - Time when output which is OFF should be turned ON
  - String in format: "hh:mm" - hours:minutes
  - Hours are in 00-23 format
  - Example: "03:00"
  - To ensure that both outputs are not turned on at the same time, this time should be greater than timeOFF

### How to setup the configuration NETIO device:

#### Import setting

**Warning:** This will overwrite all Schedules and other Actions (LUA script) which are set in the device.

1. Go to the TAB: Settings -> System
2. Select a configuration file with "Browse"
3. Click on "Import configuration"
4. Check Actions TAB if a new rule is there with correct settings
5. Restart Device

#### Manual configuration

1. Enable rule: Checked
2. Name:  (select your favourite name)
3. Description: Weekly changes of outputs X and X (user-defined)
4. Trigger: System started up
5. Schedule: Always
6. Copy the code below
7. Save and restart device (the script has 10 seconds delay after system startup to ensure stable input variables)

```LUA
--------- Configuration variables ---------
local changeDay = 2 -- [1-7] (Monday 1, Sunday 7)
local output1 = 1 -- [1-4]
local output2 = 2 -- [1-4]
local timeOFF = "10:00" -- [00:00 - 23:59]
local timeON = "10:30" -- [00:00 - 23:59]

--------- Do not change ---------
local changedON = 0
local changedOFF = 0
local reseted = 1
local outputON = output1
local outputOFF = output2
timeON = 60*60*tonumber(timeON:sub(1,2)) + 60*tonumber(timeON:sub(4,5))
timeOFF = 60*60*tonumber(timeOFF:sub(1,2)) + 60*tonumber(timeOFF:sub(4,5))

local day = tonumber(os.date("%w"))
local hour = os.date("%H")
local minute = os.date("%M")
local time = 60*60*hour+60*minute
local tmp = 0

function checkDay()
  -- Load time values
  day = tonumber(os.date("%w"))
  -- Change Sunday value to 7
  if day == 0 then
    day = 7
  end
  hour = os.date("%H")
  minute = os.date("%M")
  time = 60*60*hour+60*minute
  -- Turn output ON
  if (day == changeDay and changedON == 0 and time>=timeON) then
    logf("Turning output %d ON",outputOFF)
    -- Turn output ON
    devices.system.SetOut{output=outputOFF,value=1}
    -- Check if othter output is on, if yes, turn it off
    if (devices.system["output" ..outputON.. "_state"] == 'on') then
        devices.system.SetOut{output=outputON,value=0}
        changedOFF = 1
      end
    changedON = 1
    reseted = 0
  -- Turn output OFF
  elseif (day == changeDay and changedOFF == 0 and time >= timeOFF) then
    logf("Turning output %d OFF",outputON)
    -- Turn output OFF
    devices.system.SetOut{output=outputON,value=0}
    changedOFF = 1
    reseted = 0
  end
  -- Next day allow changes for next week
  if (day == (changeDay+1)%7 and reseted == 0) then
    log("New day,swaping output states")
    changedON = 0
    changedOFF = 0
    reseted = 1
    -- Change outputs
    tmp = outputON
    outputON = outputOFF
    outputOFF = tmp
  end
  delay(10,function() checkDay() end)
end

function startScript()
  -- Load initial states of outputs or set them if they are not valid

  -- Load initial values
  local initState1 = devices.system["output" ..output1.. "_state"]
  local initState2 = devices.system["output" ..output2.. "_state"]

-- Both are turned ON or OFF
  if ((initState1=='on' and initState2=='on') or (initState1=='off' and initState2=='off')) then
    -- Turn output1 on and output2 off
    logf("Both outputs are in the same state, turning output %d ON and output %d OFF",output1,output2)
    devices.system.SetOut{output=output1,value=1}
    devices.system.SetOut{output=output2,value=0}
    outputON = output1
    outputOFF = output2
  -- One of them is turned on, second is turned off
  else
    if initState1 == 'on' then
      outputON = output1
      outputOFF = output2
    else
      outputON = output2
      outputOFF = output1
    end
  end

  -- If its change day don't swap states again
  day = tonumber(os.date("%w"))
  -- Change Sunday value to 7
  if day == 0 then
    day = 7
  end
  hour = os.date("%H")
  minute = os.date("%M")
  time = 60*60*hour+60*minute

  if (day == changeDay and (time>=timeOFF or time>=timeON)) then
    changedON = 1
    changedOFF = 1
  end

  checkDay()
end

log("Script started")
delay(10,function() startScript() end)
```

