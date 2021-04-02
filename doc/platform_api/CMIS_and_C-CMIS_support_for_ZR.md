## CMIS and C-CMIS support for ZR on SONiC

### Overview
Common Management Interface Specification (CMIS) is defined for pluggables or on-board modules to communicate with the registers. With a clear difinition of these registers, modules can set the configurations or get the status, to achieve the basic level of monitor and control. 

CMIS is widely used on modules based on a Two-Wire-Interface (TWI), including QSFP-DD, OSFP, COBO and QSFP modules. However, new requirements emerge with the introduction of coherent optical modules, such as 400G ZR. 400G ZR is the first type of modules to require definitions on coherent optical specifications, a field CMIS does not touch on. The development of C(coherent)-CMIS aims to solve this issue. It is based on CMIS but incroporates more definitions on registers in the extended space, regarding the emerging demands on coherent optics specifications.

The scope of this work is to develop APIs for both CMIS and C-CMIS to support 400G ZR modules on SONiC.

The rest of the article will discuss the following items:

- Layered architecture to access registers
- Definition on CMIS and C-CMIS registers
- Method to read from and write to registers
- High level functions

### Layered architecture to access registers
          ---------------------------
         |   High level functions    |
          ---------------------------
                /\            ||            
                ||            \/
             ---------------------
            | Decode       Encode |
             ---------------------
                /\            ||            
                ||            \/
            ------------------------
           | read_reg     write_reg |
            ------------------------               
                /\            ||            
                ||            \/
             ---------------------
            |   Module registers  |
             ---------------------           
                

### Definition on CMIS and C-CMIS registers
-  Memory structure and mapping

The host addressable memory starts with a lower memory of 128 bytes that occupy address byte 0-127. 
Then it starts from Page 0. Each page has 128 bytes and the first byte of each page starts with an offset of 128.
Therefore, the address of a byte with page and offset is page*128 + offset.

-  Module general information pages

Sample code to define registers with module general information:

```
SFF8024_IDENTIFIER = {
          'PAGE': 0x00,
          'OFFSET': 0,
          'SIZE': 1,
          'TYPE': 'B'
}

```
-  VDM pages

Sample code to define registers with VDM information below. Note that VDM ID starting from 128 are defined in C-CMIS.

```
VDM_TYPE = {
          # VDM_ID: [VDM_NAME, DATA_TYPE, SCALE]
          1: ['Laser Age [%]', 'U16', 1],
          2: ['TEC Current [%]', 'S16', 100.0/32767],
          3: ['Laser Frequency Error [MHz]', 'S16', 10],
          4: ['Laser Temperature [C]', 'S16', 1.0/256],
          5: ['eSNR Media Input [dB]', 'U16', 1.0/256],
          6: ['eSNR Host Input [dB]', 'U16', 1.0/256],
          7: ['PAM4 Level Transition Parameter Media Input [dB]', 'U16', 1.0/256],
          8: ['PAM4 Level Transition Parameter Host Input [dB]', 'U16', 1.0/256],
          9: ['Pre-FEC BER Minimum Media Input', 'F16', 1], 
          10: ['Pre-FEC BER Minimum Host Input', 'F16', 1], 
          11: ['Pre-FEC BER Maximum Media Input', 'F16', 1], 
          12: ['Pre-FEC BER Maximum Host Input', 'F16', 1], 
          13: ['Pre-FEC BER Average Media Input', 'F16', 1], 
          14: ['Pre-FEC BER Average Host Input', 'F16', 1], 
          15: ['Pre-FEC BER Current Value Media Input', 'F16', 1],
          16: ['Pre-FEC BER Current Value Host Input', 'F16', 1],
          17: ['Errored Frames Minimum Media Input', 'F16', 1], 
          18: ['Errored Frames Minimum Host Input', 'F16', 1], 
          19: ['Errored Frames Maximum Media Input', 'F16', 1], 
          20: ['Errored Frames Minimum Host Input', 'F16', 1], 
          21: ['Errored Frames Average Media Input', 'F16', 1], 
          22: ['Errored Frames Average Host Input', 'F16', 1], 
          23: ['Errored Frames Current Value Media Input', 'F16', 1], 
          24: ['Errored Frames Current Value Host Input', 'F16', 1],
          128: ['Modulator Bias X/I [%]', 'U16', 100.0/65535],
          129: ['Modulator Bias X/Q [%]', 'U16', 100.0/65535],
          130: ['Modulator Bias Y/I [%]', 'U16', 100.0/65535],
          131: ['Modulator Bias Y/Q [%]', 'U16', 100.0/65535],
          132: ['Modulator Bias X_Phase [%]', 'U16', 100.0/65535],
          133: ['Modulator Bias Y_Phase [%]', 'U16', 100.0/65535],
          134: ['CD high granularity, short link [ps/nm]', 'S16', 1], 
          135: ['CD low granularity, long link [ps/nm]', 'S16', 20],
          136: ['DGD [ps]', 'U16', 0.01],
          137: ['SOPMD [ps^2]', 'U16', 0.01],
          138: ['PDL [dB]', 'U16', 0.1],
          139: ['OSNR [dB]', 'U16', 0.1],
          140: ['eSNR [dB]', 'U16', 0.1],
          141: ['CFO [MHz]', 'S16', 1],
          142: ['EVM_modem [%]', 'U16', 100.0/65535],
          143: ['Tx Power [dBm]', 'S16', 0.01],
          144: ['Rx Total Power [dBm]', 'S16', 0.01],
          145: ['Rx Signal Power [dBm]', 'S16', 0.01],
          146: ['SOP ROC [krad/s]', 'U16', 1],
          147: ['MER [dB]', 'U16', 0.1]
}


def get_VDM_page(port, page):
    if page not in [0x20, 0x21, 0x22, 0x23]:
        raise ValueError('Page not in VDM Descriptor range!')
    VDM_descriptor = struct.unpack('128B', read_reg(port, page, 128, 128))
    # Odd Adress VDM observable type ID, real-time monitored value in Page + 4
    VDM_typeID = VDM_descriptor[1::2]
    # Even Address
    # Bit 7-4: Threshold set ID in Page + 8, in group of 8 bytes, 16 sets/page
    # Bit 3-0: n. Monitored lane n+1 
    VDM_lane = [(elem & 0xf) for elem in VDM_descriptor[0::2]]
    VDM_thresholdID = [(elem>>4) for elem in VDM_descriptor[0::2]]
    VDM_valuePage = page+4
    VDM_thrshPage = page+8
    VDM_Page_data = {}
    for index, typeID in enumerate(VDM_typeID):
        if typeID not in Data_Type_Dict.VDM_TYPE:
            continue
        else:
            vdm_info_dict = Data_Type_Dict.VDM_TYPE[typeID]
            scale = vdm_info_dict[2]
            thrshID = VDM_thresholdID[index]
            vdm_value_raw = read_reg(port, VDM_valuePage, 128+2*index, 2)
            if vdm_info_dict[1] == 'S16':
                vdm_value = struct.unpack('>h',vdm_value_raw)[0] * scale
                vdm_thrsh_high_alarm = struct.unpack('>h', read_reg(port, VDM_thrshPage, 128+8*thrshID, 2))[0] * scale
                vdm_thrsh_low_alarm = struct.unpack('>h', read_reg(port, VDM_thrshPage, 128+8*thrshID+2, 2))[0] * scale
                vdm_thrsh_high_warn = struct.unpack('>h', read_reg(port, VDM_thrshPage, 128+8*thrshID+4, 2))[0] * scale
                vdm_thrsh_low_warn = struct.unpack('>h', read_reg(port, VDM_thrshPage, 128+8*thrshID+6, 2))[0] * scale
            elif vdm_info_dict[1] == 'U16':
                vdm_value = struct.unpack('>H',vdm_value_raw)[0] * scale
                vdm_thrsh_high_alarm = struct.unpack('>H', read_reg(port, VDM_thrshPage, 128+8*thrshID, 2))[0] * scale
                vdm_thrsh_low_alarm = struct.unpack('>H', read_reg(port, VDM_thrshPage, 128+8*thrshID+2, 2))[0] * scale
                vdm_thrsh_high_warn = struct.unpack('>H', read_reg(port, VDM_thrshPage, 128+8*thrshID+4, 2))[0] * scale
                vdm_thrsh_low_warn = struct.unpack('>H', read_reg(port, VDM_thrshPage, 128+8*thrshID+6, 2))[0] * scale
            elif vdm_info_dict[1] == 'F16':
                vdm_value_int = struct.unpack('>H',vdm_value_raw)[0]
                vdm_value = get_F16(vdm_value_int)
                vdm_thrsh_high_alarm_int = struct.unpack('>H', read_reg(port, VDM_thrshPage, 128+8*thrshID, 2))[0]
                vdm_thrsh_low_alarm_int = struct.unpack('>H', read_reg(port, VDM_thrshPage, 128+8*thrshID+2, 2))[0]
                vdm_thrsh_high_warn_int = struct.unpack('>H', read_reg(port, VDM_thrshPage, 128+8*thrshID+4, 2))[0]
                vdm_thrsh_low_warn_int = struct.unpack('>H', read_reg(port, VDM_thrshPage, 128+8*thrshID+6, 2))[0]
                vdm_thrsh_high_alarm = get_F16(vdm_thrsh_high_alarm_int)
                vdm_thrsh_low_alarm = get_F16(vdm_thrsh_low_alarm_int)
                vdm_thrsh_high_warn = get_F16(vdm_thrsh_high_warn_int)
                vdm_thrsh_low_warn = get_F16(vdm_thrsh_low_warn_int)
            else:
                continue

        if vdm_info_dict[0] not in VDM_Page_data:
            VDM_Page_data[vdm_info_dict[0]] = {VDM_lane[index]+1: [vdm_value,
                                                                   vdm_thrsh_high_alarm,
                                                                   vdm_thrsh_low_alarm,
                                                                   vdm_thrsh_high_warn,
                                                                   vdm_thrsh_low_warn]}
        else:
            VDM_Page_data[vdm_info_dict[0]][VDM_lane[index]+1] = [vdm_value,
                                                                  vdm_thrsh_high_alarm,
                                                                  vdm_thrsh_low_alarm,
                                                                  vdm_thrsh_high_warn,
                                                                  vdm_thrsh_low_warn]
    return VDM_Page_data

```
-  C-CMIS related pages (Page 30h - 3fh)

Sample code to read C-CMIS defined PMs:

```
def get_PM(port):

    write_reg_from_dict(port, Page2Fh.FREEZE_REQUEST, 128)
    time.sleep(1)

    PM_dict = {}
    rx_bits_pm = read_reg_from_dict(port, Page34h.RX_BITS_PM)
    rx_bits_subint_pm = read_reg_from_dict(port, Page34h.RX_BITS_SUB_INTERVAL_PM)
    rx_corr_bits_pm = read_reg_from_dict(port, Page34h.RX_CORR_BITS_PM)
    rx_min_corr_bits_subint_pm = read_reg_from_dict(port, Page34h.RX_MIN_CORR_BITS_SUB_INTERVAL_PM)
    rx_max_corr_bits_subint_pm = read_reg_from_dict(port, Page34h.RX_MAX_CORR_BITS_SUB_INTERVAL_PM)

    if (rx_bits_subint_pm != 0) and (rx_bits_pm != 0):
        PM_dict['preFEC_BER_cur'] = rx_corr_bits_pm*1.0/rx_bits_subint_pm
        PM_dict['preFEC_BER_min'] = rx_min_corr_bits_subint_pm*1.0/rx_bits_subint_pm
        PM_dict['preFEC_BER_max'] = rx_max_corr_bits_subint_pm*1.0/rx_bits_subint_pm

    rx_frames_pm = read_reg_from_dict(port, Page34h.RX_FRAMES_PM)
    rx_frames_subint_pm = read_reg_from_dict(port, Page34h.RX_FRAMES_SUB_INTERVAL_PM)
    rx_frames_uncorr_err_pm = read_reg_from_dict(port, Page34h.RX_FRAMES_UNCORR_ERR_PM)
    rx_min_frames_uncorr_err_subint_pm = read_reg_from_dict(port, Page34h.RX_MIN_FRAMES_UNCORR_ERR_SUB_INTERVAL_PM)
    rx_max_frames_uncorr_err_subint_pm = read_reg_from_dict(port, Page34h.RX_MIN_FRAMES_UNCORR_ERR_SUB_INTERVAL_PM)

    if (rx_frames_subint_pm != 0) and (rx_frames_pm != 0):
        PM_dict['preFEC_uncorr_frame_ratio_cur'] = rx_frames_uncorr_err_pm*1.0/rx_frames_subint_pm
        PM_dict['preFEC_uncorr_frame_ratio_min'] = rx_min_frames_uncorr_err_subint_pm*1.0/rx_frames_subint_pm
        PM_dict['preFEC_uncorr_frame_ratio_max'] = rx_max_frames_uncorr_err_subint_pm*1.0/rx_frames_subint_pm


    PM_dict['rx_cd_avg'] = read_reg_from_dict(port, Page35h.RX_AVG_CD_PM)
    PM_dict['rx_cd_min'] = read_reg_from_dict(port, Page35h.RX_MIN_CD_PM)
    PM_dict['rx_cd_max'] = read_reg_from_dict(port, Page35h.RX_MAX_CD_PM)

    PM_dict['rx_dgd_avg'] = read_reg_from_dict(port, Page35h.RX_AVG_DGD_PM)*Data_Type_Dict.PM_SCALE['RX_DGD_SCALE_PS']
    PM_dict['rx_dgd_min'] = read_reg_from_dict(port, Page35h.RX_MIN_DGD_PM)*Data_Type_Dict.PM_SCALE['RX_DGD_SCALE_PS']
    PM_dict['rx_dgd_max'] = read_reg_from_dict(port, Page35h.RX_MAX_DGD_PM)*Data_Type_Dict.PM_SCALE['RX_DGD_SCALE_PS']

    PM_dict['rx_sopmd_avg'] = read_reg_from_dict(port, Page35h.RX_AVG_SOPMD_PM)*Data_Type_Dict.PM_SCALE['RX_SOPMD_SCALE_PS2']
    PM_dict['rx_sopmd_min'] = read_reg_from_dict(port, Page35h.RX_MIN_SOPMD_PM)*Data_Type_Dict.PM_SCALE['RX_SOPMD_SCALE_PS2']
    PM_dict['rx_sopmd_max'] = read_reg_from_dict(port, Page35h.RX_MAX_SOPMD_PM)*Data_Type_Dict.PM_SCALE['RX_SOPMD_SCALE_PS2']

    PM_dict['rx_pdl_avg'] = read_reg_from_dict(port, Page35h.RX_AVG_PDL_PM)*Data_Type_Dict.PM_SCALE['RX_PDL_SCALE_DB']
    PM_dict['rx_pdl_min'] = read_reg_from_dict(port, Page35h.RX_MIN_PDL_PM)*Data_Type_Dict.PM_SCALE['RX_PDL_SCALE_DB']
    PM_dict['rx_pdl_max'] = read_reg_from_dict(port, Page35h.RX_MAX_PDL_PM)*Data_Type_Dict.PM_SCALE['RX_PDL_SCALE_DB']

    PM_dict['rx_osnr_avg'] = read_reg_from_dict(port, Page35h.RX_AVG_OSNR_PM)*Data_Type_Dict.PM_SCALE['OSNR_SCALE_DB']
    PM_dict['rx_osnr_min'] = read_reg_from_dict(port, Page35h.RX_MIN_OSNR_PM)*Data_Type_Dict.PM_SCALE['OSNR_SCALE_DB']
    PM_dict['rx_osnr_max'] = read_reg_from_dict(port, Page35h.RX_MAX_OSNR_PM)*Data_Type_Dict.PM_SCALE['OSNR_SCALE_DB']

    PM_dict['rx_esnr_avg'] = read_reg_from_dict(port, Page35h.RX_AVG_ESNR_PM)*Data_Type_Dict.PM_SCALE['ESNR_SCALE_DB']
    PM_dict['rx_esnr_min'] = read_reg_from_dict(port, Page35h.RX_MIN_ESNR_PM)*Data_Type_Dict.PM_SCALE['ESNR_SCALE_DB']
    PM_dict['rx_esnr_max'] = read_reg_from_dict(port, Page35h.RX_MAX_ESNR_PM)*Data_Type_Dict.PM_SCALE['ESNR_SCALE_DB']

    PM_dict['rx_cfo_avg'] = read_reg_from_dict(port, Page35h.RX_AVG_CFO_PM)
    PM_dict['rx_cfo_min'] = read_reg_from_dict(port, Page35h.RX_MIN_CFO_PM)
    PM_dict['rx_cfo_max'] = read_reg_from_dict(port, Page35h.RX_MAX_CFO_PM)

    PM_dict['rx_evm_avg'] = read_reg_from_dict(port, Page35h.RX_AVG_EVM_PM)*Data_Type_Dict.PM_SCALE['EVM_SCALE']
    PM_dict['rx_evm_min'] = read_reg_from_dict(port, Page35h.RX_MIN_EVM_PM)*Data_Type_Dict.PM_SCALE['EVM_SCALE']
    PM_dict['rx_evm_max'] = read_reg_from_dict(port, Page35h.RX_MAX_EVM_PM)*Data_Type_Dict.PM_SCALE['EVM_SCALE']

    PM_dict['tx_power_avg'] = read_reg_from_dict(port, Page35h.TX_AVG_POWER_PM)*Data_Type_Dict.PM_SCALE['POWER_SCALE_DBM']
    PM_dict['tx_power_min'] = read_reg_from_dict(port, Page35h.TX_MIN_POWER_PM)*Data_Type_Dict.PM_SCALE['POWER_SCALE_DBM']
    PM_dict['tx_power_max'] = read_reg_from_dict(port, Page35h.TX_MAX_POWER_PM)*Data_Type_Dict.PM_SCALE['POWER_SCALE_DBM']

    PM_dict['rx_power_avg'] = read_reg_from_dict(port, Page35h.RX_AVG_POWER_PM)*Data_Type_Dict.PM_SCALE['POWER_SCALE_DBM']
    PM_dict['rx_power_min'] = read_reg_from_dict(port, Page35h.RX_MIN_POWER_PM)*Data_Type_Dict.PM_SCALE['POWER_SCALE_DBM']
    PM_dict['rx_power_max'] = read_reg_from_dict(port, Page35h.RX_MAX_POWER_PM)*Data_Type_Dict.PM_SCALE['POWER_SCALE_DBM']

    PM_dict['rx_sigpwr_avg'] = read_reg_from_dict(port, Page35h.RX_AVG_SIG_POWER_PM)*Data_Type_Dict.PM_SCALE['POWER_SCALE_DBM']
    PM_dict['rx_sigpwr_min'] = read_reg_from_dict(port, Page35h.RX_MIN_SIG_POWER_PM)*Data_Type_Dict.PM_SCALE['POWER_SCALE_DBM']
    PM_dict['rx_sigpwr_max'] = read_reg_from_dict(port, Page35h.RX_MAX_SIG_POWER_PM)*Data_Type_Dict.PM_SCALE['POWER_SCALE_DBM']

    PM_dict['rx_soproc_avg'] = read_reg_from_dict(port, Page35h.RX_AVG_SIG_SOPROC_PM)
    PM_dict['rx_soproc_min'] = read_reg_from_dict(port, Page35h.RX_MIN_SIG_SOPROC_PM)
    PM_dict['rx_soproc_max'] = read_reg_from_dict(port, Page35h.RX_MAX_SIG_SOPROC_PM)

    PM_dict['rx_mer_avg'] = read_reg_from_dict(port, Page35h.RX_AVG_MER_PM)*Data_Type_Dict.PM_SCALE['MER_SCALE_DB']
    PM_dict['rx_mer_min'] = read_reg_from_dict(port, Page35h.RX_MIN_MER_PM)*Data_Type_Dict.PM_SCALE['MER_SCALE_DB']
    PM_dict['rx_mer_max'] = read_reg_from_dict(port, Page35h.RX_MAX_MER_PM)*Data_Type_Dict.PM_SCALE['MER_SCALE_DB']

    write_reg_from_dict(port, Page2Fh.FREEZE_REQUEST, 0)
    return PM_dict
```

### Method to read from and write to registers

#### Read and write registers
- read_reg
- write_reg

Sample code to read and write registers with vendor provided functoins:

```
import sonic_platform.platform
import sonic_platform_base.sonic_sfp.sfputilhelper
platform_chassis = sonic_platform.platform.Platform().get_chassis()
PAGE_SIZE = 128

def read_reg(port, page, offset, size):
    # platform_chassis.get_sfp(port).write_eeprom(127,1,bytearray([page]))
    return platform_chassis.get_sfp(port).read_eeprom(page*PAGE_SIZE + offset,size)

def write_reg(port, page, offset, size, write_raw):
    # platform_chassis.get_sfp(port).write_eeprom(127,1,bytearray([page]))
    platform_chassis.get_sfp(port).write_eeprom(page*PAGE_SIZE + offset,size,write_raw)
    
```
#### Encoding and decoding raw data
- read_reg_from_dict
- write_reg_from_dict

Sample code to decode and encode from/to rawdata in registers:

```
def read_reg_from_dict(port, Dict): 
    read_raw = read_reg(port, page = Dict['PAGE'], offset = Dict['OFFSET'], size = Dict['SIZE'])
    read_buffer = struct.unpack(Dict['TYPE'], read_raw)
    if len(read_buffer) == 1:
        return read_buffer[0]
    else:
        return read_buffer

def write_reg_from_dict(port, Dict, write_buffer):
    write_raw = struct.pack(Dict['TYPE'], write_buffer)
    write_reg(port, page = Dict['PAGE'], offset = Dict['OFFSET'], size = Dict['SIZE'], write_raw = write_raw)
  
```

### High level functions

#### Get module basic information
- get_module_type
- get_module_status
- get_module_vendor
- get_module_part_number
- get_module_serial_number
- get_datapath_lane_status
- get_module_case_temp
- get_supply_3v3
- get_laser_temp
- get_tuning_status
- get_laser_freq
- get_TX_configured_power

#### Get VDM related information
- get_VDM

```
def get_VDM(port):
    vdm_page_supported_raw = read_reg_from_dict(port, Page2Fh.VDM_SUPPORT) & 0x3
    vdm_page_supported = Data_Type_Dict.VDM_SUPPORTED_PAGE[vdm_page_supported_raw]
    VDM = {}
    # Bit 7, freeze all PMs for reporting
    write_reg_from_dict(port, Page2Fh.FREEZE_REQUEST, 128)
    time.sleep(1)
    for page in vdm_page_supported:
        VDM_current_page = get_VDM_page(port, page)
        VDM.update(VDM_current_page)
    write_reg_from_dict(port, Page2Fh.FREEZE_REQUEST, 0)
    return VDM
```

#### Get C-CMIS PM
- get_PM

#### Set module configuration, turn up
- set_low_power

```
def set_low_power(port, AssertLowPower):
    module_control = AssertLowPower << 6    
    write_reg_from_dict(port, Page00h_Lower.MODULE_LEVEL_CONTROL, module_control)

```
- set_TX_power

```
def set_TX_power(port, TX_power):
    
    # Target programmable output power increments of 0.01 dBm.
    TX_POWER_SCALE = 0.01
    TX_power_buffer = round(TX_power / TX_POWER_SCALE)
    write_reg_from_dict(port, Page12h.TX_TARGET_OUTPUT_POWER_LANE_1,TX_power_buffer)

    # Keep reading tuning in progress bit until it completes. TX_TUNING_STATUS_LANE_1
    # Bit 1: 0b, TX tuning not in progress; 1b, TX tuning in progress (both freq and power)
    # Bit 0: TX wavelength unlocked real-time status. 0b, wavelength locked; 1b, wavelength unlocked
    WAIT_TIME = 30 # wait for 30 seconds
    counter = 0
    while counter < WAIT_TIME:
        tuning_status = read_reg_from_dict(port, Page12h.TX_TUNING_STATUS_LANE_1) & 0x3
        if (tuning_status == 0):
            break
        time.sleep(1)
        counter += 1

    tuning_status = read_reg_from_dict(port, Page12h.TX_TUNING_STATUS_LANE_1) & 0x3
    if (tuning_status == 0):  # Tuning complete
        print('Success! Tuning completes!')
    else:
        print('Error! Tuning failed!')
```
- set_laser_freq

```
def set_laser_freq(port, freq):
    # Note the input argument freq must be in THz
    set_low_power(port, True)
    time.sleep(5)
    # Set selected grid spacing in Page 12h to 75 GHz
    # Bit 7-4: TX lane {n} selected grid spacing RW
    # Selected grid spacing of lane n=1-8.
    # 0000b: 3.125 GHz
    # 0001b: 6.25 GHz
    # 0010b: 12.5 GHz
    # 0011b: 25 GHz
    # 0100b: 50 GHz
    # 0101b: 100 GHz
    # 0110b: 33 GHz
    # 0111: 75 GHz
    # 8-14: Reserved
    # 1111b: Not available

    # Bit 0: TX lane {n} fine tuning enable RW
    # Bool fine-tuning enabled for lane n=1-8.
    # 0b: Fine-tuning disabled
    # 1b: Fine-tuning enabled.
    freq_grid = 0x70
    write_reg_from_dict(port, Page12h.TX_TUNING_SETTING_LANE_1, freq_grid)

    # Channel number n for 75 GHz grid spacing
    # Frequency (THz) = 193.1 + n * 0.025, where n must be divisible by 3.
    channel_number = round((freq - 193.1)/0.025)
    if channel_number % 3 is not 0:
        print('Error! Frequency has to be in 75 GHz grid!')
        return
    write_reg_from_dict(port, Page12h.TX_CHANNEL_NUM_LANE_1, channel_number)

    # Deassert low power mode
    set_low_power(port, False)
    
    # Keep reading tuning in progress bit until it completes. TX_TUNING_STATUS_LANE_1
    # Bit 1: 0b, TX tuning not in progress; 1b, TX tuning in progress (both freq and power)
    # Bit 0: TX wavelength unlocked real-time status. 0b, wavelength locked; 1b, wavelength unlocked
    WAIT_TIME = 30 # wait for 30 seconds
    counter = 0
    while counter < WAIT_TIME:
        tuning_status = read_reg_from_dict(port, Page12h.TX_TUNING_STATUS_LANE_1) & 0x3
        if (tuning_status == 0):
            break
        time.sleep(1)
        counter += 1

    tuning_status = read_reg_from_dict(port, Page12h.TX_TUNING_STATUS_LANE_1) & 0x3
    if (tuning_status == 0):  # Tuning complete
        print('Success! Tuning completes!')
    else:
        print('Error! Tuning failed!')
```