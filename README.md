# FeitCSI Project

Replicating the FeitCSI project which is an open-source 802.11 CSI (Channel State Information) extraction and frame injection tool for Intel wireless NICs.

## Quick Links
- [FeitCSI: Official Documentation](https://feitcsi.kuskosoft.com/)
- [CSIKit Integration](https://github.com/Gi-z/CSIKit)

## Tested Environment
- **Laptop**: Dell XPS 13 7390
- **OS**: Ubuntu 24.04.3 LTS (Noble)
- **WiFi Card**: Intel Corporation Wi-Fi 6 AX200 (rev 1a)

## Installation

### Prerequisites
Install required build dependencies:
```bash
sudo apt update && sudo apt install -y flex bison
sudo apt install -y iw
```

### Installation Steps
The `.deb` files are located in the `/debs` folder. 

1. Install the iwlwifi driver package first:
   ```bash
   sudo dpkg -i debs/feitcsi-iwlwifi_2.0.0_all.deb
   ```

2. Install the main FeitCSI package:
   ```bash
   sudo dpkg -i debs/feitcsi_2.0.0_all.deb
   ```

**Note:** The `feitcsi-iwlwifi_2.0.0_all.deb` installation may fail if `flex` and `bison` are not installed, as the DKMS build process requires these tools for compiling the kernel module.

### Setting up CSIKit for Data Analysis

Create a Python virtual environment and install CSIKit for analyzing the collected CSI data:

```bash
python3 -m venv venv
source venv/bin/activate
pip install csikit
```

## Usage

1. Check your wireless interfaces:
   ```bash
   iw dev
   iw dev wlp2s0 info
   ```
   The `iw dev wlp2s0 info` command provides the `--frequency` and `--channel-width` values needed for the next command.

2. Start CSI collection:
   ```bash
   sudo feitcsi --frequency 5180 --channel-width 80 --format HESU --output-file data/<name>.dat -v
   ```

3. Analyze the data with CSIKit:
   ```bash
   csikit --graph data/<name>.dat
   ```

4. Alternatively, use the provided Jupyter notebook for data analysis:
   ```bash
   jupyter lab notebooks/parseFeitCSI.ipynb
   ```

## Technical Details

### CSI Functionality
The package adds CSI (Channel State Information) extraction through:
- Vendor commands: `IWL_MVM_VENDOR_CMD_CSI_EVENT` (`iwl-vendor-cmd.h`)
- CSI data attributes: `IWL_MVM_VENDOR_ATTR_CSI_HDR` and `IWL_MVM_VENDOR_ATTR_CSI_DATA` (`iwl-vendor-cmd.h`)
- Notification handlers: `CSI_HEADER_NOTIFICATION` and `CSI_CHUNKS_NOTIFICATION` (`location.h`)

**File Locations:**
- `iwl-vendor-cmd.h`: `/usr/src/feitcsi-iwlwifi-2.0.0/drivers/net/wireless/intel/iwlwifi/iwl-vendor-cmd.h`
- `location.h`: `/usr/src/feitcsi-iwlwifi-2.0.0/drivers/net/wireless/intel/iwlwifi/fw/api/location.h`
 
These files are included in the `feitcsi-iwlwifi_2.0.0_all.deb` package inside `/debs` folder, which can be extracted (for inspection) as follows:

```bash
mkdir -p src && cd src && mkdir -p feitcsi-iwlwifi_2.0.0_all \
   && dpkg-deb -x ../debs/feitcsi-iwlwifi_2.0.0_all.deb feitcsi-iwlwifi_2.0.0_all \
   && dpkg-deb -e ../debs/feitcsi-iwlwifi_2.0.0_all.deb feitcsi-iwlwifi_2.0.0_all/DEBIAN
```
> **Note:** The `src/` directory is included in `.gitignore` and must be added manually if needed.

### Data Flow
1. **CSI Collection**: The firmware collects CSI data from received packets
2. **Chunked Transfer**: Large CSI data is split into chunks (max 16 chunks)
3. **Netlink Interface**: Data is sent to userspace via vendor-specific netlink messages
4. **Data Format**: Each CSI measurement includes:
   - Header with metadata (timestamp, RSSI, rate info, antenna config)
   - Complex CSI matrix data (subcarriers x RX antennas x TX antennas)

### Data Structure
Each CSI measurement contains:
- **272-byte header** with metadata including:
  - Timestamp and FTM clock
  - Source MAC address
  - Rate format (CCK, OFDM, HT, VHT, HE, EHT)
  - Channel width (20/40/80/160/360 MHz)
  - RSSI values
  - Antenna configuration
  - Number of subcarriers, TX/RX antennas
- **Variable-length CSI data**: Complex numbers (16-bit real + 16-bit imaginary) for each subcarrier-antenna pair

## Worth Exploring

- https://github.com/seemoo-lab/nexmon_csi/issues/207
- https://www.linuxquestions.org/questions/linux-networking-3/measure-channel-state-information-csi-using-iwlwifi-4175726176/