# Evolvable Enums

Microsoft Graph API Design Pattern

*The evolvable enum pattern allows API producers to extend enumerated types with new members without breaking API consumers.*

## Problem

---
Frequently API producers want to add new members to an enum type after it is initially published. Some serialization libraries may fail when they encounter members in an enum type that were added after the serialization model was generated. Within this documented we will refer to any added enum members as unknown.

## Solution

---
The solution here is to add a 'sentinel' member named ```unknownFutureValue``` at the end of the currently known enum members. The API producer will then replace any member that is numerically after ```unknownFutureValue``` with ```unknownFutureValue```. If an API consumer can handle unknown enum values the consumer can opt into receiving the unknown enum members by specifying the ```Prefer: include-unknown-enum-members``` HTTP Header in their request(s), the API producer will then indicate that this preference has been applied by returning the ```Preference-Applied: include-unknown-enum-members``` HTTP header in the response.

## When to Use this Pattern

---

- It is a best practice to include a ```unknownFutureValue``` value when the enum is initially introduced to allow flexibility to extend the enum during the lifetime of the API. Even if the API producer believes they have included all possible members in an enum it is still strongly recommended to include a ```unknownFutureValue``` member to allow for unforeseen future circumstances which may require extending the enum.

- This pattern must not be used in scenarios where an API consumer wants to use enum members that are not known to the API producer.

## Issues and Considerations

---

- An enum member with the name of ```unknownFutureValue``` **must** only be used as a sentinel value, an API producer **must** not include a member named ```unknownFutureValue``` in an enum for any other purpose.

- Changing the value (i.e. position) of the ```unknownFutureValue``` sentinel member is considered a breaking change, and must follow the [deprecation](../deprecation.md) process.

- Enum Types can have multiple members with the same numeric value to allow for aliasing enum members, ```unknownFutureValue``` **must** not be aliased to any other enum member.

- There is no ability for a client to indicate that it can handle a subset of unknown enum members, instead that can only specify either that they can not handle any unknown enum members, or they can handle any unknown enum members.

- The ```Prefer: include-unknown-enum-members``` header applies to all included enums in the request/response, there is no way for an API consumer to apply the behavior to only a subset of enum types.

- New values **must** not be inserted into the enum before ```unknownFutureValue```, implementers are recommended to make the numeric value of ```unknownFutureValue``` one greater than the last known enum member to ensure there are no gaps into which a new member could be inadvertently added. The exception to this is the case of flagged enums in which case the value of ```unknownFutureValue``` should be be next power of 2 value.

- For flagged enums care should be exercised to ensure that ```unknownFutureValue``` is not included in any enum members that represent a combination of other enum members.

- If the value of a property containing a flag enum contains multiple unknown values they should all be replaced with a single ```unknownFutureValue``` value (i.e. there should not have multiple ```unknownFutureValue``` values returned).

- If an API consumer specifies ```unknownFutureValue``` for the value of a property in a ```POST```/```PUT``` request or as parameter of an action or function the API producer must reject the request with a ```400 Bad Request``` HTTP status.

- If an API consumer specifies ```unknownFutureValue``` for the value of a property in a ```PATCH``` request the API producer must treat the property as if it were absent (i.e. the existing value should not be changed). For the case where the API producer treats ```PATCH``` as an upsert the call **must** be rejected with a ```400 Bad Request``` HTTP status.

- If an API consumer specifies an enum member greater than ```unknownFutureValue``` in any request without specifying the ```Prefer: include-unknown-enum-members``` header the API producer must reject the request with a ```400 Bad Request``` HTTP status.

- For details of how the ```unknownFutureValue``` value is handled as part of a ```$filter``` clause please consult the following examples:

### CSDL

```xml
<EntityType Name="example">
    <Property Name="enumProperty" Type="exampleEnum"/>
</EntityType>

<EnumType Name="exampleEnum">
    <Member Name="default" Value="0"/>
    <Member Name="one" Value="1"/>
    <Member Name="unknownFutureValue" Value="2"/>
    <Member Name="newValue" Value="3"/>
</EnumType>
```

### Filter Behavior

| ```$filter``` clause | ```Prefer: include-unknown-enum-members``` Absent | ```Prefer: include-unknown-enum-members``` Present |
|---|---|---|
| ```enumProperty eq unknownFutureValue```| Return entities where enumProperty has any value greater than ```unknownFutureValue``` replacing actual value with ```unknownFutureValue```| Return nothing |
| ```enumProperty gt unknownFutureValue```| Return entities where enumProperty has any value greater than ```unknownFutureValue``` replacing actual value with ```unknownFutureValue``` | Return entities where enumProperty has any value greater than ```unknownFutureValue``` |
| ```enumProperty lt unknownFutureValue```| Return entities where enumProperty has any known value (i.e. less than ```unknownFutureValue```) | Return entities where enumProperty has any value less than ```unknownFutureValue```|
| ```enumProperty eq newValue``` | ```400 Bad Request``` | Return entities where enumProperty has the value ```newValue``` |
| ```enumProperty gt newValue``` | ```400 Bad Request``` | Return entities where enumProperty has a value greater than ```newValue``` |
| ```enumProperty lt newValue``` | ```400 Bad Request``` | Return entities where enumProperty has a value less than ```newValue``` |

- If an evolvable enum is included in an ```$orderby``` clause the actual numeric value of the member should be used to order the collection, after sorting the member should then be replaced with ```unknownFutureValue``` when the ```Prefer: include-unknown-enum-members``` header is absent.

## Examples

---

For the following examples we will consider the ```managedDevice``` entity which refers to the ```managedDeviceArchitecture``` enum type.

```xml
<!-- Simplified entity for example purposes -->
<EntityType Name="managedDevice" BaseType="graph.entity">
    <Property Name="displayName" Type="Edm.String" />
    <Property Name="processorArchitecture" Type="graph.managedDeviceArchitecture"/>
</EntityType>
```

When the ```managedDeviceArchitecture``` enum was initially published to Graph it was defined as below:

```xml
<!-- Slightly modified enum for example purposes -->
<EnumType Name="managedDeviceArchitecture">
    <Member Name="unknown" Value="0"/>
    <Member Name="x86" Value="1"/>
    <Member Name="x64" Value="2"/>
    <Member Name="arm" Value="3"/>
    <Member Name="arm64" Value="4"/>
    <Member Name="unknownFutureValue" Value="5"/>
</EnumType>
```

The enum was later extended to add a new value of ```quantum``` leading to the below CSDL

```xml
<EnumType Name="managedDeviceArchitecture">
    <Member Name="unknown" Value="0"/>
    <Member Name="x86" Value="1"/>
    <Member Name="x64" Value="2"/>
    <Member Name="arm" Value="3"/>
    <Member Name="arm64" Value="4"/>
    <Member Name="unknownFutureValue" Value="5"/>
    <Member Name="quantum" Value="6"/>
</EnumType>
```

### Default Behavior

```http
GET https://graph.microsoft.com/v1.0/deviceManagement/managedDevices?$select=displayName,processorArchitecture
```

```json
{
    "value": [
        { 
            "id": "0", 
            "displayName": "Surface Pro X", 
            "processorArchitecture" : "arm64"
        },
        { 
            "id": "1",
            "displayName": "Prototype",
            "processorArchitecture": "unknownFutureValue"
        }
        { 
            "id": "2",
            "displayName": "My Laptop", 
            "processorArchitecture": "x64"
        }
    ]
}
```

In this case the value of the ```processorArchitecture``` property is ```quantum``` however since the client did not request the ```include-unknown-enum-members``` header the value was replaced with ```unknownFutureValue```

### Include opt-in Header

```http
GET https://graph.microsoft.com/v1.0/deviceManagement/managedDevices?$select=displayName,processorArchitecture

Prefer: include-unknown-enum-members
```

```json
Preference-Applied: include-unknown-enum-members

{
    "value": [
        { 
            "displayName": "Surface Pro X", 
            "processorArchitecture" : "arm64"
        },
        { 
            "displayName": "Prototype",
            "processorArchitecture": "quantum"
        },
        { 
            "displayName": "My Laptop",
            "processorArchitecture": "x64"
        }
    ]
}
```

### Default Sort Behavior

```http
GET https://graph.microsoft.com/v1.0/deviceManagement/managedDevices?$select=displayName,processorArchitecture&$orderBy=processorArchitecture
```

```json
{
    "value": [
        { 
            "displayName": "My Laptop", 
            "processorArchitecture": "x64"
        },
        { 
            "displayName": "Surface Pro X", 
            "processorArchitecture" : "arm64"
        },
        { 
            "displayName": "Prototype",
            "processorArchitecture": "unknownFutureValue"
        }
    ]
}
```

### Sort Behavior with opt-in Header

```http
GET https://graph.microsoft.com/v1.0/deviceManagement/managedDevices?$select=displayName,processorArchitecture&$orderBy=processorArchitecture

Prefer: include-unknown-enum-members
```

```json
Preference-Applied: include-unknown-enum-members

{
    "value": [
        { 
            "displayName": "My Laptop",
            "processorArchitecture": "x64"
        },
        { 
            "displayName": "Surface Pro X",
            "processorArchitecture" : "arm64"
        },
        { 
            "displayName": "Prototype", 
            "processorArchitecture": "quantum"
        }
    ]
}
```

### Default Filter Behavior

```http
GET https://graph.microsoft.com/v1.0/deviceManagement/managedDevices?$select=displayName,processorArchitecture&$filter=processorArchitecture gt x64
```

```json
{
    "value": [
        { 
            "displayName": "My Laptop", 
            "processorArchitecture": "x64"
        },
        { 
            "displayName": "Prototype",
            "processorArchitecture": "unknownFutureValue"
        }
    ]
}
```

### Filter Behavior with opt-in Header

```http
GET https://graph.microsoft.com/v1.0/deviceManagement/managedDevices?$select=displayName,processorArchitecture&$filter=processorArchitecture gt x64

Prefer: include-unknown-enum-members
```

```json
Preference-Applied: include-unknown-enum-members

{
    "value": [
        { 
            "displayName": "My Laptop", 
            "processorArchitecture": "x64"
        },
        { 
            "displayName": "Prototype", 
            "processorArchitecture": "quantum"
        }
    ]
}
```

### Patch Example

```http
PATCH https://graph.microsoft.com/v1.0/deviceManagement/managedDevices/1

{ 
    "displayName": "Secret Prototype",
    "processorArchitecture": "unknownFutureValue"
}
```

```json
{
    "id": "1",
    "displayName": "Secret Prototype",
    "processorArchitecture": "unknownFutureValue"
}
```

```http
GET https://graph.microsoft.com/v1.0/deviceManagement/managedDevices/1
Prefer: include-unknown-enum-members
```

```json
Preference-Applied: include-unknown-enum-members

{
    "id": "1",
    "displayName": "Secret Prototype",
    "processorArchitecture": "quantum"
}
```

## Flag Enum Examples

For the following examples we will consider the ```windowsUniversalAppX``` entity which refers to the ```windowsArchitecture``` flag enum type.

```xml
<!-- Simplified entity for example purposes -->
<EntityType Name="windowsUniversalAppX" BaseType="graph.entity">
    <Property Name="displayName" Type="Edm.String" />
    <Property Name="applicableArchitectures" Type="graph.windowsArchitecture"/>
</EntityType>
```

When the ```windowsArchitecture``` enum was initially published to Graph it was defined as below:

```xml
<!-- Slightly modified enum for example purposes -->
<EnumType Name="windowsArchitecture" IsFlags="true">
    <Member Name="none" Value="0"/>
    <Member Name="x86" Value="1"/>
    <Member Name="x64" Value="2"/>
    <Member Name="arm" Value="4"/>
    <Member Name="neutral" Value="8"/>
    <Member Name="unknownFutureValue" Value="16" />
</EnumType>
```

The enum was later extended to add a new value of ```quantum``` leading to the below CSDL

```xml
<EnumType Name="windowsArchitecture" IsFlags="true">
    <Member Name="none" Value="0"/>
    <Member Name="x86" Value="1"/>
    <Member Name="x64" Value="2"/>
    <Member Name="arm" Value="4"/>
    <Member Name="neutral" Value="8"/>
    <Member Name="unknownFutureValue" Value="16" />
    <Member Name="quantum" Value="32" />
</EnumType>
```

### Flag Enum Default Behavior

```http
GET https://graph.microsoft.com/v1.0/deviceAppManagement/mobileApps?$select=displayName,applicableArchitectures
```

```json
{
    "value": [
        { 
            "id": "0", 
            "displayName": "OneNote", 
            "applicableArchitectures" : "neutral"
        },
        { 
            "id": "1",
            "displayName": "Minecraft",
            "applicableArchitectures": "x86,x64,arm,unknownFutureValue"
        }
        { 
            "id": "2",
            "displayName": "Edge", 
            "applicableArchitectures": "x64,arm,unknownFutureValue"
        }
    ]
}
```

In this case the value of the ```applicableArchitectures``` property includes ```quantum``` however since the client did not request the ```include-unknown-enum-members``` header the value was replaced with ```unknownFutureValue```

### Flag Enum Include opt-in Header

```http
GET https://graph.microsoft.com/v1.0/deviceAppManagement/mobileApps?$select=displayName,applicableArchitectures

Prefer: include-unknown-enum-members
```

```json
Preference-Applied: include-unknown-enum-members

{
    "value": [
        { 
            "id": "0", 
            "displayName": "OneNote", 
            "applicableArchitectures" : "neutral"
        },
        { 
            "id": "1",
            "displayName": "Minecraft",
            "applicableArchitectures": "x86,x64,arm,quantum"
        }
        { 
            "id": "2",
            "displayName": "Edge", 
            "applicableArchitectures": "x64,arm,quantum"
        }
    ]
}
```

### Flag Enum Default Filter Behavior

```http
GET https://graph.microsoft.com/v1.0/deviceAppManagement/mobileApps?$select=displayName,applicableArchitectures&$filter=applicableArchitectures has unknownFutureValue
```

```json
{
    "value": [
        { 
            "id": "1",
            "displayName": "Minecraft",
            "applicableArchitectures": "x86,x64,arm,unknownFutureValue"
        }
        { 
            "id": "2",
            "displayName": "Edge", 
            "applicableArchitectures": "x64,arm,unknownFutureValue"
        }
    ]
}
```

### Flag Enum Include opt-in Header Filter Behavior

```http
GET https://graph.microsoft.com/v1.0/deviceAppManagement/mobileApps?$select=displayName,applicableArchitectures&$filter=applicableArchitectures has unknownFutureValue

Prefer: include-unknown-enum-members
```

```json
Preference-Applied: include-unknown-enum-members

{
    "value": []
}
```

### Flag Enum Patch Example

```http
PATCH https://graph.microsoft.com/v1.0/deviceAppManagement/mobileApps/1

{ 
    "displayName": "Minecraft 2",
    "processorArchitecture": "unknownFutureValue"
}
```

```json
{
    "id": "1",
    "displayName": "Minecraft 2",
    "applicableArchitectures": "unknownFutureValue"
}
```

```http
GET https://graph.microsoft.com/v1.0/deviceAppManagement/mobileApps/1

Prefer: include-unknown-enum-members
```

```json
Preference-Applied: include-unknown-enum-members

{
    "id": "1",
    "displayName": "Minecraft 2",
    "applicableArchitectures": "x86,x64,arm,quantum"
}
```
