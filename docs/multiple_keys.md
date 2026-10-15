# MCUboot Multiple Key Support

## Overview

MCUboot now supports verifying images signed with multiple keys. This feature allows the bootloader to accept images signed by any of the configured keys, which is useful for:

- **Key Rotation**: Gradually migrate from one signing key to another
- **Multiple Sources**: Support images from different vendors or teams
- **Backup Keys**: Maintain backup keys for disaster recovery
- **Development and Production**: Use different keys for different environments

## Background

Previously, MCUboot only supported a single signing key. While upstream MCUboot has the capability to handle multiple keys, Zephyr's integration only exposed support for one key. This implementation adds Kconfig options and build system support to enable multiple keys in Zephyr-based MCUboot.

## Configuration

### Kconfig Options

To enable multiple key support, use the following Kconfig options:

```kconfig
CONFIG_BOOT_MULTIPLE_SIGNATURE_KEYS=y
CONFIG_BOOT_SIGNATURE_KEY_FILE="path/to/key1.pem"
CONFIG_BOOT_SIGNATURE_KEY_FILE_2="path/to/key2.pem"
```

### Configuration Details

- `CONFIG_BOOT_MULTIPLE_SIGNATURE_KEYS`: Enable multiple key support (default: n)
- `CONFIG_BOOT_SIGNATURE_KEY_FILE`: Primary signing key (required)
- `CONFIG_BOOT_SIGNATURE_KEY_FILE_2`: Second signing key (required when multiple keys enabled)

### Dependencies

Multiple key support requires:
- A signature type other than `BOOT_SIGNATURE_TYPE_NONE`
- Hardware key support must be disabled (`!BOOT_HW_KEY`)
- Key matching bypass must be disabled (`!BOOT_BYPASS_KEY_MATCH`)

## Usage

### Basic Example

1. Generate multiple signing keys:

```bash
cd bootloader/mcuboot
python3 scripts/imgtool.py keygen -k key1.pem -t ecdsa-p256
python3 scripts/imgtool.py keygen -k key2.pem -t ecdsa-p256
```

2. Configure MCUboot with multiple keys in `mcuboot.conf`:

```
CONFIG_BOOT_MULTIPLE_SIGNATURE_KEYS=y
CONFIG_BOOT_SIGNATURE_KEY_FILE="key1.pem"
CONFIG_BOOT_SIGNATURE_KEY_FILE_2="key2.pem"
```

3. Build your application with sysbuild:

```bash
west build -b <board> <app> --sysbuild
```

4. Sign different images with different keys:

```bash
# Sign image with key1
west sign -t imgtool -- --key key1.pem

# Sign another image with key2
west sign -t imgtool -- --key key2.pem
```

MCUboot will verify and boot images signed with any of the configured keys.

### Key Rotation Example

To rotate from an old key to a new key:

1. Add the new key as a second key while keeping the old key as primary
2. Deploy the updated bootloader
3. Start signing new images with the new key
4. After all devices are updated, make the new key primary and remove the old key

## Implementation Details

### Build System

The CMake build system (`bootloader/mcuboot/boot/zephyr/CMakeLists.txt`) processes each configured key file:

1. Resolves the key file path (absolute or relative)
2. Runs `imgtool.py getpub` to extract the public key
3. Generates C source files (`autogen-pubkey.c`, `autogen-pubkey-2.c`, etc.)
4. Compiles these files into the bootloader

### Key Array

The `bootutil_keys` array in `keys.c` is sized to support 2 keys:

```c
const struct bootutil_key bootutil_keys[2] = {
    { .key = ecdsa_pub_key, .len = &ecdsa_pub_key_len },
    { .key = ecdsa_pub_key_2, .len = &ecdsa_pub_key_2_len },
};
const int bootutil_key_cnt = 2;
```

### Verification Process

During image verification, MCUboot:
1. Reads the image signature
2. Iterates through all keys in `bootutil_keys`
3. Attempts verification with each key
4. Accepts the image if any key successfully verifies the signature

## Testing

A comprehensive test suite is available at `openair/tests/boot/test_mcuboot_multikey/`:

```bash
west build -b <board> openair/tests/boot/test_mcuboot_multikey --sysbuild
west flash
```

The test creates multiple applications signed with different keys and verifies that MCUboot can boot any of them.

## Security Considerations

- **Key Management**: Store private keys securely and limit access
- **Key Revocation**: This implementation does not support key revocation; all configured keys are trusted equally
- **Maximum Keys**: Limit the number of keys to minimize attack surface and boot time
- **Production Keys**: Never use the default test keys in production

## Limitations

- Maximum of 2 keys supported
- All keys must use the same signature algorithm (RSA, ECDSA-P256, or ED25519)
- No key revocation mechanism
- Both key slots must have valid keys when multiple key support is enabled

## References

- MCUboot documentation: https://docs.mcuboot.com/
- Zephyr MCUboot integration: https://docs.zephyrproject.org/latest/services/device_mgmt/dfu/mcuboot.html

