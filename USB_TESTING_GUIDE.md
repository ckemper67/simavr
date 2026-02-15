# Testing USB Communication with simavr in GoogleTest

## simavr USB Architecture

The USB peripheral simulation lives in `simavr/sim/avr_usb.c` and exposes
everything through `avr_ioctl()` calls. There is no need for the Linux VHCI
subsystem -- that's only for bridging to a real host. You can act as a
**virtual USB host** directly in your test code.

## Key API: `avr_io_usb` and IOCTLs

The data exchange struct (from `simavr/sim/avr_usb.h`):

```cpp
struct avr_io_usb {
    uint8_t pipe;    // endpoint number (bit 7 = direction)
    uint32_t sz;     // buffer size / actual bytes transferred
    uint8_t *buf;    // data buffer
};
```

Available IOCTLs:

| IOCTL | Purpose |
|-------|---------|
| `AVR_IOCTL_USB_SETUP` | Send a SETUP packet to control endpoint |
| `AVR_IOCTL_USB_READ` | Read data from device (IN transfer) |
| `AVR_IOCTL_USB_WRITE` | Write data to device (OUT transfer) |
| `AVR_IOCTL_USB_RESET` | Signal USB bus reset |
| `AVR_IOCTL_USB_VBUS` | Signal VBUS power on/off |

Return values: `0` = success, `AVR_IOCTL_USB_NAK` (-2) = not ready,
`AVR_IOCTL_USB_STALL` (-3) = stalled.

## Approach for GoogleTest

You act as the USB host, just like `vhci_usb.c` does but without the Linux
kernel bridge. The pattern is:

```cpp
#include "sim_avr.h"
#include "avr_usb.h"

class UsbTest : public ::testing::Test {
protected:
    avr_t *avr;

    void SetUp() override {
        // ... load firmware, init avr ...
        // Power on VBUS
        avr_ioctl(avr, AVR_IOCTL_USB_VBUS, (void*)1);
        // Wait for firmware to attach (run cycles)
        // Reset the bus
        avr_ioctl(avr, AVR_IOCTL_USB_RESET, NULL);
        // Run more cycles for firmware to handle reset
    }

    // Send a control read request (e.g. GET_DESCRIPTOR)
    int controlRead(uint8_t reqtype, uint8_t req,
                    uint16_t wValue, uint16_t wIndex,
                    uint16_t wLength, uint8_t *data) {
        // 1. Send SETUP packet
        struct usbsetup {
            uint8_t rt; uint8_t r;
            uint16_t wV, wI, wL;
        } __attribute__((packed));
        usbsetup setup = {reqtype, req, wValue, wIndex, wLength};
        avr_io_usb pkt = {0, sizeof(setup), (uint8_t*)&setup};
        avr_ioctl(avr, AVR_IOCTL_USB_SETUP, &pkt);

        // 2. Run AVR cycles so firmware processes it
        runCycles(10000);

        // 3. Read response
        pkt.pipe = 0;
        pkt.sz = wLength;
        pkt.buf = data;
        int ret;
        for (int i = 0; i < 100; i++) {
            ret = avr_ioctl(avr, AVR_IOCTL_USB_READ, &pkt);
            if (ret != AVR_IOCTL_USB_NAK) break;
            runCycles(1000);
        }
        return (ret == 0) ? pkt.sz : ret;
    }

    // Read from a bulk/interrupt IN endpoint
    int bulkRead(uint8_t ep, uint8_t *data, uint32_t len) {
        avr_io_usb pkt = {ep, len, data};
        int ret;
        for (int i = 0; i < 100; i++) {
            ret = avr_ioctl(avr, AVR_IOCTL_USB_READ, &pkt);
            if (ret != AVR_IOCTL_USB_NAK) break;
            runCycles(1000);
        }
        return (ret == 0) ? pkt.sz : ret;
    }

    void runCycles(int n) {
        for (int i = 0; i < n; i++)
            avr_run(avr);
    }
};

TEST_F(UsbTest, GetDeviceDescriptor) {
    uint8_t desc[18];
    int len = controlRead(0x80, 6 /*GET_DESCRIPTOR*/,
                          0x0100 /*DEVICE*/, 0, 18, desc);
    ASSERT_EQ(len, 18);
    EXPECT_EQ(desc[1], 1); // bDescriptorType == DEVICE
}
```

## Detecting Attach

You can monitor when the firmware attaches to the bus via the IRQ:

```cpp
avr_irq_t *irq = avr_io_getirq(avr, AVR_IOCTL_USB_GETIRQ(), USB_IRQ_ATTACH);
avr_irq_register_notify(irq, [](avr_irq_t*, uint32_t value, void* param) {
    bool *attached = (bool*)param;
    *attached = !!value;
}, &device_attached);
```

## Important Caveats

1. **Thread safety** -- the IOCTLs are not thread-safe (noted as TODO in the
   code). Since you're driving everything from your test, this shouldn't be an
   issue as long as you interleave `avr_run()` and IOCTL calls in the same
   thread.

2. **NAK polling** -- the firmware may NAK if it hasn't processed data yet. You
   need to run AVR cycles between retries (the `vhci_usb.c` uses `usleep()`
   because it runs the AVR in a separate thread).

3. **5 endpoints max** (0-4), each with 64-byte FIFO buffers. Dual-bank
   buffering is not fully implemented.

4. **No SOF generation** -- Start-of-Frame interrupts are not automatically
   generated (TODO in the code). If your firmware depends on SOF timing, you may
   need to work around this.

## USB Register Map (for reference)

The USB controller uses these hardware registers (offsets from `r_usbcon`):

| Offset | Register | Purpose |
|--------|----------|---------|
| 0 | `usbcon` | USB control |
| 8 | `udcon` | USB device control |
| 9 | `udint` | USB device interrupt flags |
| 10 | `udien` | USB device interrupt enables |
| 11 | `udaddr` | USB device address |
| 12-13 | `udfnuml/h` | Frame number |
| 16 | `ueintx` | Endpoint interrupt flags (per-endpoint) |
| 17 | `uenum` | Endpoint number select |
| 19 | `ueconx` | Endpoint control (per-endpoint) |
| 20-21 | `uecfg0x/1x` | Endpoint config (per-endpoint) |
| 25 | `uedatx` | Endpoint data register (per-endpoint) |
| 26 | `uebclx` | Endpoint byte count (per-endpoint) |

## Summary

You don't need VHCI at all. Just call `avr_ioctl()` directly as a virtual USB
host, interleaving with `avr_run()` to let the firmware execute.
