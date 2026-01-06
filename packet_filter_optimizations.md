# Packet Filter Optimization Recommendations

## Summary
The packet filtering diff is fundamentally sound but needs optimization for embedded constraints.

## Critical Fixes

### 1. Eliminate Code Duplication (Flash Memory)

**Current**: type_names array defined 4 times (~500 bytes wasted)

**Fix**: Add to CommonCLI.h or a constants header:

```cpp
// In CommonCLI.h or new file helpers/PacketTypeConstants.h
#define MAX_PACKET_TYPES 16

namespace mesh {
  // Use PROGMEM on ESP32 to keep in flash, not RAM
  #ifdef ESP32_PLATFORM
  const char PTYPE_NAME_0[] PROGMEM = "req";
  const char PTYPE_NAME_1[] PROGMEM = "resp";
  const char PTYPE_NAME_2[] PROGMEM = "txt";
  const char PTYPE_NAME_3[] PROGMEM = "ack";
  const char PTYPE_NAME_4[] PROGMEM = "advert";
  const char PTYPE_NAME_5[] PROGMEM = "grp.txt";
  const char PTYPE_NAME_6[] PROGMEM = "grp.data";
  const char PTYPE_NAME_7[] PROGMEM = "anon";
  const char PTYPE_NAME_8[] PROGMEM = "path";
  const char PTYPE_NAME_9[] PROGMEM = "trace";
  const char PTYPE_NAME_10[] PROGMEM = "multi";
  const char PTYPE_NAME_11[] PROGMEM = "control";
  const char PTYPE_NAME_12[] PROGMEM = "12";
  const char PTYPE_NAME_13[] PROGMEM = "13";
  const char PTYPE_NAME_14[] PROGMEM = "14";
  const char PTYPE_NAME_15[] PROGMEM = "raw";

  const char* const PACKET_TYPE_NAMES[MAX_PACKET_TYPES] PROGMEM = {
    PTYPE_NAME_0, PTYPE_NAME_1, PTYPE_NAME_2, PTYPE_NAME_3,
    PTYPE_NAME_4, PTYPE_NAME_5, PTYPE_NAME_6, PTYPE_NAME_7,
    PTYPE_NAME_8, PTYPE_NAME_9, PTYPE_NAME_10, PTYPE_NAME_11,
    PTYPE_NAME_12, PTYPE_NAME_13, PTYPE_NAME_14, PTYPE_NAME_15
  };
  #else
  // For non-ESP32, simpler approach
  static const char* const PACKET_TYPE_NAMES[MAX_PACKET_TYPES] = {
    "req", "resp", "txt", "ack", "advert", "grp.txt", "grp.data",
    "anon", "path", "trace", "multi", "control", "12", "13", "14", "raw"
  };
  #endif

  // Helper function to get type name (handles PROGMEM on ESP32)
  inline const char* getPacketTypeName(uint8_t type, char* buf) {
    if (type >= MAX_PACKET_TYPES) return "unknown";
    #ifdef ESP32_PLATFORM
      strcpy_P(buf, (char*)pgm_read_ptr(&PACKET_TYPE_NAMES[type]));
      return buf;
    #else
      return PACKET_TYPE_NAMES[type];
    #endif
  }
}
```

**Savings**: ~400 bytes flash, eliminates 3 duplicate arrays

### 2. Fix Buffer Overflow Protection

**Current issue**: Doesn't account for commas properly

**Fix** in CommonCLI.cpp config display:
```cpp
} else if (memcmp(config, "no.repeat", 9) == 0) {
  reply[0] = '>';
  reply[1] = ' ';
  int pos = 2;
  bool has_any = false;
  char name_buf[16]; // For PROGMEM reads on ESP32

  for (int i = 0; i < MAX_PACKET_TYPES; i++) {
    if ((_prefs->no_repeat_packet_types & (1 << i)) != 0) {
      const char* name = mesh::getPacketTypeName(i, name_buf);
      int len = strlen(name);

      // Ensure space for: current string + comma + name + null terminator
      if (pos + (has_any ? 1 : 0) + len + 1 >= 160) {
        // Truncate gracefully
        strcpy(&reply[pos], "...");
        pos += 3;
        break;
      }

      if (has_any) {
        reply[pos++] = ',';
      }
      strcpy(&reply[pos], name);
      pos += len;
      has_any = true;
    }
  }

  if (!has_any) {
    strcpy(&reply[2], "none");
  } else {
    reply[pos] = 0;
  }
}
```

### 3. Optimize Advert Parsing

**Current**: Creates AdvertDataParser for every filtered advert

**Optimization**: Check advert type earlier, only parse if needed

```cpp
uint8_t pkt_type = packet->getPayloadType();
if (pkt_type < MAX_PACKET_TYPES && (_prefs.no_repeat_packet_types & (1 << pkt_type)) != 0) {

  // Special case: allow companion adverts through even if adverts filtered
  if (pkt_type == PAYLOAD_TYPE_ADVERT) {
    const int app_data_offset = PUB_KEY_SIZE + 4 + SIGNATURE_SIZE;

    // Quick sanity check before parsing
    if (packet->payload_len > app_data_offset) {
      const uint8_t* app_data = &packet->payload[app_data_offset];
      int app_data_len = packet->payload_len - app_data_offset;

      // Only parse if there's actual data
      if (app_data_len > 0) {
        // Quick check: is first byte valid flags? (avoid full parse if malformed)
        uint8_t flags = app_data[0];
        uint8_t adv_type = flags & 0x0F;

        if (adv_type == ADV_TYPE_CHAT) {
          MESH_DEBUG_PRINTLN("allowPacketForward: allowing companion advert");
          return true;
        }
      }
    }
  }

  MESH_DEBUG_PRINTLN("allowPacketForward: type %d filtered", pkt_type);
  return false;
}
```

**Improvement**: Avoids creating AdvertDataParser unless type is ADV_TYPE_CHAT

### 4. Add Validation on Load

**Add** to CommonCLI.cpp in loadPrefsInt():
```cpp
file.read((uint8_t *)&_prefs->no_repeat_packet_types, sizeof(_prefs->no_repeat_packet_types));

// Validate: ensure no bits set beyond valid packet types
uint16_t valid_mask = (1 << MAX_PACKET_TYPES) - 1;
_prefs->no_repeat_packet_types &= valid_mask;
```

### 5. Add Missing Include

**Add** to MyMesh.h (if not already present):
```cpp
#include <helpers/AdvertDataHelpers.h>
```

## Additional Considerations

### 1. Network Impact
Filtering packet types can fragment the network if not configured consistently:
- **CONTROL packets**: Should generally NOT be filtered (critical for discovery)
- **TRACE packets**: Filtering prevents path discovery
- **ACK packets**: Filtering breaks reliability

**Recommendation**: Add warnings in CLI or documentation

### 2. Companion Advert Logic
The special handling for ADV_TYPE_CHAT is good, but consider:
- Should other advert types (like sensors) also be allowed?
- Make this configurable?

### 3. Default Configuration
```cpp
_prefs.no_repeat_packet_types = 0x0000;  // Repeat everything by default
```

**Consider**: Providing sensible defaults for high-traffic networks:
```cpp
// Example: Filter group messages and raw custom by default
// _prefs.no_repeat_packet_types = (1 << PAYLOAD_TYPE_GRP_TXT) | (1 << PAYLOAD_TYPE_GRP_DATA);
```

## Performance Impact

### Current Implementation
- **Flash**: +~600 bytes (with duplication)
- **RAM**: +2 bytes (uint16_t in prefs)
- **CPU**: Minimal - single bit check per packet
- **Stack**: ~80 bytes when parsing adverts

### After Optimizations
- **Flash**: +~200 bytes (eliminates duplication, uses PROGMEM)
- **RAM**: +2 bytes (same)
- **CPU**: Same or better (faster advert check)
- **Stack**: ~16 bytes (avoids AdvertDataParser creation)

## Testing Recommendations

1. **Test with all types filtered**: Ensure network remains functional
2. **Test companion adverts**: Verify ADV_TYPE_CHAT still works when adverts filtered
3. **Test buffer overflow**: Set all 16 types and check `config no.repeat` output
4. **Test persistence**: Verify settings survive reboot
5. **Test multi-platform**: ESP32, NRF52, RP2040, STM32

## Conclusion

✅ **Concept is excellent** - addresses real problem of network flooding
⚠️ **Implementation needs refinement** - mainly code organization issues
🔧 **Optimizations will improve** - flash usage, safety, and performance

**Overall Assessment**: With the suggested optimizations, this is a valuable feature for MeshCore that maintains embedded development best practices.
