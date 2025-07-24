# LQX-20 Deployment Guide

## Quick Start

### Windows Deployment

#### Prerequisites
```powershell
# Install Visual Studio 2019 or later with C++ tools
# Install CMake 3.16 or later
# Install Git for Windows

# Verify installations
cmake --version
cl.exe /?
```

#### Build and Install
```powershell
# Clone and build
git clone https://github.com/lackadaisical-security/lqx10.git
cd lqx10
mkdir build
cd build

# Configure
cmake .. -G "Visual Studio 16 2019" -A x64

# Build
cmake --build . --config Release

# Test
ctest -C Release

# Install
cmake --build . --config Release --target install
```

### Linux Deployment

#### Prerequisites (Ubuntu/Debian)
```bash
# Install dependencies
sudo apt update
sudo apt install build-essential cmake git

# For development
sudo apt install valgrind cppcheck clang-tools
```

#### Prerequisites (RHEL/CentOS)
```bash
# Install dependencies
sudo yum groupinstall "Development Tools"
sudo yum install cmake3 git

# For development  
sudo yum install valgrind cppcheck clang-analyzer
```

#### Build and Install
```bash
# Clone and build
git clone https://github.com/lackadaisical-security/lqx10.git
cd lqx10
mkdir build
cd build

# Configure
cmake .. -DCMAKE_BUILD_TYPE=Release

# Build
make -j$(nproc)

# Test
ctest

# Install
sudo make install
```

### macOS Deployment

#### Prerequisites
```bash
# Install Xcode command line tools
xcode-select --install

# Install Homebrew
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

# Install dependencies
brew install cmake git
```

#### Build and Install
```bash
# Clone and build
git clone https://github.com/lackadaisical-security/lqx10.git
cd lqx10
mkdir build
cd build

# Configure
cmake .. -DCMAKE_BUILD_TYPE=Release

# Build
make -j$(sysctl -n hw.ncpu)

# Test
ctest

# Install
sudo make install
```

## Production Deployment

### System Requirements

#### Minimum Requirements
- **CPU**: x86-64 processor with AES-NI support
- **Memory**: 1 GB RAM
- **Storage**: 500 MB available space
- **OS**: Windows 10, Linux (kernel 4.0+), macOS 10.15+

#### Recommended Requirements
- **CPU**: Modern x86-64 with hardware acceleration
- **Memory**: 4 GB RAM
- **Storage**: 2 GB available space (for logging and temporary files)
- **Network**: Reliable internet connection

### Configuration

#### Security Configuration
```c
// Production security settings
lqx10_config_t config = {
    .key_schedule_rounds = 100000,      // High iteration count
    .layer_mask = 0x3FF,                // All 10 layers enabled
    .runtime_mutation_enabled = true,   // Enable anti-analysis
    .anti_debug_enabled = true,         // Enable debug protection
    .entropy_pool_size = 4096,          // Large entropy pool
    .constant_time_operations = true    // Side-channel protection
};
```

#### Performance Configuration
```c
// Balanced performance settings
lqx10_config_t config = {
    .key_schedule_rounds = 50000,       // Moderate iteration count
    .layer_mask = 0x1FF,                // Skip final shroud (layer 10)
    .runtime_mutation_enabled = false,  // Disable for performance
    .anti_debug_enabled = false,        // Disable for debugging
    .entropy_pool_size = 1024,          // Smaller entropy pool
    .hardware_acceleration = true       // Use AES-NI if available
};
```

### Environment Setup

#### Production Environment
```bash
# Set environment variables
export LQX10_CONFIG_PATH="/etc/lqx10/lqx10.conf"
export LQX10_LOG_LEVEL="WARNING"
export LQX10_ENTROPY_SOURCE="hardware"

# Create configuration directory
sudo mkdir -p /etc/lqx10
sudo mkdir -p /var/log/lqx10
sudo mkdir -p /var/lib/lqx10

# Set permissions
sudo chmod 755 /etc/lqx10
sudo chmod 755 /var/log/lqx10
sudo chmod 700 /var/lib/lqx10
```

#### Development Environment
```bash
# Set environment variables
export LQX10_CONFIG_PATH="./lqx10.conf"
export LQX10_LOG_LEVEL="DEBUG"
export LQX10_ENTROPY_SOURCE="software"
export LQX10_DEBUG_MODE="1"
```

## Integration Guide

### C/C++ Applications

#### Basic Integration
```c
#include <lqx10_core.h>

int main() {
    // Initialize context
    lqx10_context_t *ctx = NULL;
    lqx10_error_t result = lqx10_init(&ctx);
    if (result != LQX10_SUCCESS) {
        fprintf(stderr, "Failed to initialize LQX10: %s\n", 
                lqx10_error_string(result));
        return 1;
    }
    
    // Derive key from password
    const char *password = "SecurePassword123!";
    uint8_t salt[32];
    lqx10_secure_random_bytes(salt, sizeof(salt));
    
    result = lqx10_key_derive(ctx, 
                             (uint8_t*)password, strlen(password),
                             salt, sizeof(salt), 
                             100000);
    if (result != LQX10_SUCCESS) {
        lqx10_destroy(ctx);
        return 1;
    }
    
    // Encrypt data
    const char *plaintext = "Secret message";
    uint8_t ciphertext[1024];
    size_t ciphertext_len = sizeof(ciphertext);
    
    result = lqx10_encrypt(ctx, 
                          (uint8_t*)plaintext, strlen(plaintext),
                          ciphertext, &ciphertext_len);
    if (result != LQX10_SUCCESS) {
        lqx10_destroy(ctx);
        return 1;
    }
    
    printf("Encryption successful, ciphertext length: %zu\n", ciphertext_len);
    
    // Cleanup
    lqx10_destroy(ctx);
    return 0;
}
```

#### Advanced Integration
```c
#include <lqx10_core.h>
#include <lqx10_mfa.h>

// Application-specific configuration
typedef struct {
    lqx10_context_t *crypto_ctx;
    uint8_t totp_secret[32];
    char *config_file;
    bool debug_mode;
} app_context_t;

// Initialize application
int app_init(app_context_t *app, const char *config_file) {
    // Load configuration
    app->config_file = strdup(config_file);
    app->debug_mode = false;
    
    // Initialize LQX10
    lqx10_error_t result = lqx10_init(&app->crypto_ctx);
    if (result != LQX10_SUCCESS) {
        return -1;
    }
    
    // Configure security settings
    lqx10_config_t config = {
        .key_schedule_rounds = 100000,
        .layer_mask = 0x3FF,
        .runtime_mutation_enabled = true,
        .anti_debug_enabled = true
    };
    
    result = lqx10_context_configure(app->crypto_ctx, &config);
    if (result != LQX10_SUCCESS) {
        lqx10_destroy(app->crypto_ctx);
        return -1;
    }
    
    // Generate TOTP secret
    lqx10_secure_random_bytes(app->totp_secret, sizeof(app->totp_secret));
    
    return 0;
}

// Encrypt with MFA
int app_encrypt_with_mfa(app_context_t *app, 
                        const uint8_t *plaintext, size_t plaintext_len,
                        uint8_t *ciphertext, size_t *ciphertext_len,
                        uint32_t totp_code) {
    // Verify TOTP
    lqx10_error_t result = lqx10_totp_verify_current(app->totp_secret, 
                                                    sizeof(app->totp_secret),
                                                    totp_code);
    if (result != LQX10_SUCCESS) {
        return -1; // Invalid TOTP
    }
    
    // Perform encryption
    return lqx10_encrypt(app->crypto_ctx, plaintext, plaintext_len,
                        ciphertext, ciphertext_len) == LQX10_SUCCESS ? 0 : -1;
}
```

### Python Integration (via ctypes)

```python
import ctypes
import os
from ctypes import *

# Load LQX10 library
if os.name == 'nt':
    lqx10 = ctypes.CDLL('./lqx10.dll')
else:
    lqx10 = ctypes.CDLL('./liblqx10.so')

# Define structures
class LQX10Context(Structure):
    _fields_ = [("opaque", c_void_p)]

# Define function prototypes
lqx10.lqx10_init.argtypes = [POINTER(POINTER(LQX10Context))]
lqx10.lqx10_init.restype = c_int

lqx10.lqx10_destroy.argtypes = [POINTER(LQX10Context)]
lqx10.lqx10_destroy.restype = c_int

lqx10.lqx10_encrypt.argtypes = [POINTER(LQX10Context), 
                               POINTER(c_ubyte), c_size_t,
                               POINTER(c_ubyte), POINTER(c_size_t)]
lqx10.lqx10_encrypt.restype = c_int

# Python wrapper class
class LQX10:
    def __init__(self):
        self.ctx = POINTER(LQX10Context)()
        result = lqx10.lqx10_init(byref(self.ctx))
        if result != 0:
            raise Exception(f"Failed to initialize LQX10: {result}")
    
    def __del__(self):
        if hasattr(self, 'ctx') and self.ctx:
            lqx10.lqx10_destroy(self.ctx)
    
    def encrypt(self, plaintext):
        plaintext_bytes = (c_ubyte * len(plaintext))(*plaintext)
        ciphertext_len = c_size_t(len(plaintext) * 12)  # Estimate overhead
        ciphertext = (c_ubyte * ciphertext_len.value)()
        
        result = lqx10.lqx10_encrypt(self.ctx, plaintext_bytes, len(plaintext),
                                    ciphertext, byref(ciphertext_len))
        if result != 0:
            raise Exception(f"Encryption failed: {result}")
        
        return bytes(ciphertext[:ciphertext_len.value])

# Usage example
crypto = LQX10()
plaintext = b"Hello, World!"
ciphertext = crypto.encrypt(plaintext)
print(f"Ciphertext length: {len(ciphertext)}")
```

## Monitoring and Maintenance

### Logging Configuration

#### Log Levels
- **ERROR**: Critical errors that prevent operation
- **WARNING**: Non-critical issues that should be addressed
- **INFO**: General operational information
- **DEBUG**: Detailed debugging information (development only)

#### Log File Setup (Linux)
```bash
# Configure rsyslog
echo "local0.*    /var/log/lqx10/lqx10.log" >> /etc/rsyslog.conf
systemctl restart rsyslog

# Configure log rotation
cat > /etc/logrotate.d/lqx10 << EOF
/var/log/lqx10/*.log {
    weekly
    rotate 52
    compress
    delaycompress
    missingok
    notifempty
    create 644 root root
}
EOF
```

### Performance Monitoring

#### Key Metrics
- **Throughput**: Operations per second
- **Latency**: Average/percentile response times  
- **Memory Usage**: Peak and average memory consumption
- **CPU Usage**: Processor utilization
- **Error Rate**: Failed operations percentage

#### Monitoring Script
```bash
#!/bin/bash
# lqx10_monitor.sh

LOG_FILE="/var/log/lqx10/performance.log"
INTERVAL=60  # seconds

while true; do
    TIMESTAMP=$(date '+%Y-%m-%d %H:%M:%S')
    
    # Get process statistics
    PID=$(pgrep lqx10cli)
    if [ -n "$PID" ]; then
        CPU=$(ps -p $PID -o %cpu --no-headers)
        MEM=$(ps -p $PID -o %mem --no-headers)
        echo "$TIMESTAMP,CPU:$CPU,MEM:$MEM" >> $LOG_FILE
    fi
    
    sleep $INTERVAL
done
```

### Security Monitoring

#### Intrusion Detection
```bash
# Monitor for suspicious activity
tail -f /var/log/lqx10/lqx10.log | grep -E "(CRITICAL|ERROR)" | \
while read line; do
    echo "ALERT: $line" | mail -s "LQX10 Security Alert" admin@company.com
done
```

#### File Integrity Monitoring
```bash
# Create checksums for binaries
find /usr/local/bin -name "lqx10*" -exec sha256sum {} \; > /var/lib/lqx10/checksums.txt

# Verify integrity (run periodically)
sha256sum -c /var/lib/lqx10/checksums.txt || {
    echo "ALERT: Binary integrity check failed!" | \
    mail -s "LQX10 Integrity Alert" admin@company.com
}
```

## Troubleshooting

### Common Issues

#### Build Problems
```bash
# Missing dependencies
sudo apt install build-essential cmake git

# CMake version too old
wget https://cmake.org/files/v3.20/cmake-3.20.0-linux-x86_64.sh
sudo sh cmake-3.20.0-linux-x86_64.sh --prefix=/usr/local --skip-license

# Compiler errors
export CC=gcc-9
export CXX=g++-9
```

#### Runtime Issues
```bash
# Permission denied
sudo chmod +x /usr/local/bin/lqx10cli

# Missing libraries
ldd /usr/local/bin/lqx10cli
export LD_LIBRARY_PATH=/usr/local/lib:$LD_LIBRARY_PATH

# Core dumps
ulimit -c unlimited
gdb /usr/local/bin/lqx10cli core
```

#### Performance Issues
```bash
# Enable hardware acceleration
echo 1 > /proc/sys/kernel/acpi_debug_level  # Enable AES-NI logs
grep -i aes /proc/cpuinfo  # Check AES-NI support

# Memory issues
valgrind --tool=memcheck --leak-check=full ./lqx10cli test

# CPU profiling
perf record ./lqx10cli benchmark
perf report
```

### Debugging

#### Debug Build
```bash
# Build with debug symbols
cmake .. -DCMAKE_BUILD_TYPE=Debug -DLQX10_DEBUG=ON
make -j$(nproc)

# Run with debugger
gdb ./lqx10cli
(gdb) run test
(gdb) bt
```

#### Memory Debugging
```bash
# Address Sanitizer
cmake .. -DCMAKE_BUILD_TYPE=Debug -DSANITIZE_ADDRESS=ON
make -j$(nproc)
./lqx10_tests

# Valgrind
valgrind --tool=memcheck --leak-check=full --show-leak-kinds=all ./lqx10_tests
```

### Support

For technical support and bug reports:
- **Email**: support@lackadaisical.dev
- **GitHub Issues**: https://github.com/lackadaisical-security/lqx10/issues
- **Documentation**: https://docs.lackadaisical.dev/lqx10
- **Security Issues**: security@lackadaisical.dev (GPG key available)

Include the following information in support requests:
- Operating system and version
- LQX10 version
- Build configuration
- Relevant log files
- Steps to reproduce the issue
