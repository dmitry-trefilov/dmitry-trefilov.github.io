---
layout: post
comments: true
title: "You validated a new DDR4 memory for NXP Layerscape board in CodeWarrior. What next?"
excerpt: "It's not enough just to insert the generated code into the ATF (ARM Trusted Firmware)"
date: 2024-12-21 10:00:00
tags: layerscape, codewarrior, ddr, nxp, ls1046
---

[![](/assets/images/centering_the_clock_passed.png)](/assets/images/centering_the_clock_passed.png)

Let's say, you have some evaluation Layerscape board from NXP, or your custom board based on this CPU. You validated a new DDR4 memory using CodeWarrior ARM V8 ISA v11.5.12, and obtained the generated code. Of course, you have already:
- read the [QCVS DDR Tool User Guide](https://www.nxp.com/docs/en/user-guide/QCVS_DDR_User_Guide.pdf)
- watched the webinar named [QorIQ® DDR Configuration and QCVS Tool](https://www.nxp.com/company/about-nxp/smarter-world-videos/QOIQ-DDR-CONFIGURATION-QCVS)
- read the blog post [DDR Controller Configuration on LS2085/LS2080 Bringing up](https://community.nxp.com/t5/Qonverge-Knowledge-Base/DDR-Controller-Configuration-on-LS2085-LS2080-Bringing-up/ta-p/1128310?attachment-id=14775)
- read the chapter `5.2.1.1 TF-A DDR Driver` in the [Layerscape Linux Distribution POC User Guide](https://www.nxp.com/docs/en/user-guide/LLDPUG_RevL6.1.1-1.0.0.pdf).

You put the generated code into the ATF recipe in your Yocto Kirkstone project, rebuild it under default options (DDR training is enabled, no static configuration), flash it to QSPI or eMMC, and... the board doesn't boot!

```sh
NOTICE:  UDIMM 99U5713-015.A00G

NOTICE:  4 GB DDR4, 64-bit, CL=15, ECC off
NOTICE:  BL2: v2.4(release):lf-5.10.52-2.1.0-rc2-0-gbb4957067-dirty
NOTICE:  BL2: Built : 04:45:39, Sep  8 2021
ERROR:   BL2: Failed to load image id 3 (-2)
Authentication failure

```

Why? The answer is non-obvious: the layout of data structures in the CodeWarrior's generated code is distinct from the sources in [nxp/qoriq-atf at the corresponding commit](https://github.com/nxp-qoriq/atf/tree/lf-5.10.52-2.1.0).

Let's look at the generated code in CodeWarrior.

ddr_init1.c:
```c
/*
**
**     
**     Copyright 2018-2021 NXP
**     
**      SPDX-License-Identifier: BSD-3-Clause
**     
**     
*/

#include <platform_def1.h>
#include <ddr1.h>

#ifdef CONFIG_STATIC_DDR
const struct ddr_cfg_regs static_2100 = {
    .cs[0].bnds = 0xFF,
    .cs[0].config = 0x80010412,
    .cs[0].config_2 = 0x00,
    .cs[1].config_2 = 0x00,
    .cs[2].config_2 =  0x00,
    .cs[3].config_2 =  0x00,
    .timing_cfg[0] = 0xFA770018,
    .timing_cfg[1] = 0xE2EAA065,
    .timing_cfg[2] = 0x005951A0,
    .timing_cfg[3] = 0x02101100,
    .timing_cfg[4] = 0x00220002,
    .timing_cfg[5] = 0x04401400,
    .timing_cfg[7] = 0x26600000,
    .timing_cfg[8] = 0x04446A00,
    .sdram_cfg[0] = 0x45200000,
    .sdram_cfg[1] = 0x00401050,
    .dq_map[0] = 0x5B633558,
    .dq_map[1] = 0xD8CD56D8,
    .dq_map[2] = 0x3355B630,
    .dq_map[3] = 0xD4000000,
    .sdram_mode[0] = 0x01010425,
    .sdram_mode[1] = 0x00100000,
    .sdram_mode[2] = 0x00,
    .sdram_mode[3] = 0x00,
    .sdram_mode[4] = 0x00,
    .sdram_mode[5] = 0x00,
    .sdram_mode[6] = 0x00,
    .sdram_mode[7] = 0x00,
    .sdram_mode[8] = 0x0500,
    .sdram_mode[9] = 0x08A40000,
    .sdram_mode[10] = 0x00,
    .sdram_mode[11] = 0x00,
    .sdram_mode[12] = 0x00,
    .sdram_mode[13] = 0x00,
    .sdram_mode[14] = 0x00,
    .sdram_mode[15] = 0x00,
    .md_cntl = 0x00,
    .interval = 0x1FFE07FF,
    .data_init = 0xDEADBEEF,
    .clk_cntl = 0x02400000,
    .init_addr = 0x00,
    .ddr_sr_cntr = 0x0,
    .init_ext_addr = 0x00,
    .zq_cntl = 0x8A090705,
    .wrlvl_cntl[0] = 0x86550608,
    .wrlvl_cntl[1] = 0x0709090D,
    .wrlvl_cntl[2] = 0x0D0F0F0C,
    .cdr[0] = 0x80080000,
    .cdr[1] = 0x80,
};


long long board_static_ddr(struct ddr_info *priv)
{
        memcpy(&priv->ddr_reg, &static_2100, sizeof(static_2100));

        return 0x80000000;
}

#else

static const struct rc_timing rcz[] = {
    {2100, 9, 8},
    {}
};

static const struct board_timing ram[] = {
    {0x02, rcz, 0x0709090D, 0x0D0F0F0C},
};

int ddr_board_options(struct ddr_info *priv)
{
        int ret;
        struct memctl_opt *popts = &priv->opt;

        ret = cal_board_params(priv, ram, ARRAY_SIZE(ram));
        if (ret)
                return ret;

        popts->bstopre = 0;
        popts->half_strength_drive_en = 1;
        popts->cpo_sample = 0x46;
        popts->ddr_cdr1 = 0x80080000;
        popts->ddr_cdr2 = 0x80;
        popts->output_driver_impedance = 1;
        popts->addr_hash = 1; /* address hashing */
        return 0;
}

/*
* Below structure is generated for DIMM part number: 99U5713-015.A00G
* Sample code to bypass reading SPD. 
* This is a sample, not recommended  for boards with slots. 
*/
struct dimm_params ddr_raw_timing = {
        .n_ranks = 1,
        .rank_density = 0x0000000100000000u,
        .capacity = 0x0000000100000000u,
        .primary_sdram_width = 64,
        .ec_sdram_width = 0,
        .device_width = 8,
        .die_density = 0x4,
        .rdimm = 0,
        .mirrored_dimm = 0,
        .n_row_addr = 16,
        .n_col_addr = 10,
        .bank_addr_bits = 0,
        .bank_group_bits = 2,
        .edc_config = 0,
        .burst_lengths_bitmask = 0x0c,
        .tckmin_x_ps = 625,
        .tckmax_ps = 1600,
        .caslat_x = 0x00FFFC00,
        .taa_ps = 13750,
        .trcd_ps = 13750,
        .trp_ps = 13750,
        .tras_ps = 32000,
        .trc_ps = 45750,
        .twr_ps = 15000,
        .trfc1_ps = 350000,
        .trfc2_ps = 260000,
        .trfc4_ps = 160000,
        .tfaw_ps = 30000,
        .trrds_ps = 5300,
        .trrdl_ps = 6400,
        .tccdl_ps = 5000,
        .refresh_rate_ps = 7800000,
        .dq_mapping[0] = 0x5B633558,
        .dq_mapping[1] = 0xD8CD56D8,
        .dq_mapping[2] = 0x3355B630,
        .dq_mapping[3] = 0xD4000000,
        .dq_mapping[4] = 0x0,
        .dq_mapping_ors = 0,
        .rc = 0x1C
};

int ddr_get_ddr_params(struct dimm_params *pdimm,
                            struct ddr_conf *conf)
{
    static const char dimm_model[] = "Fixed DDR on board";
        conf->dimm_in_use[0] = 1;       /* Modify accordingly */
        memcpy(pdimm, &ddr_raw_timing, sizeof(struct dimm_params));
        memcpy(pdimm->mpart, dimm_model, sizeof(dimm_model) - 1);

        return 1;
}
#endif /* CONFIG_DDR_NODIMM */


long long _init_ddr(void)
{
    int spd_addr[] = { NXP_SPD_EEPROM0 };
    struct ddr_info info;
    struct sysinfo sys;
    long long dram_size;

    zeromem(&sys, sizeof(sys));
    get_clocks(&sys);

    zeromem(&info, sizeof(struct ddr_info));
    info.num_ctlrs = 1;
    info.dimm_on_ctlr = 1;
    info.clk = get_ddr_freq(&sys, 0);
    info.spd_addr = spd_addr;
    info.ddr[0] = (void *)NXP_DDR_ADDR;

    dram_size = dram_init(&info);

    if (dram_size < 0)
        ERROR("DDR init failed.\n");

    erratum_a008850_post();
    return dram_size;
}
```

<details>
  <summary>Click to expand platform_def1.h
</summary>

```c
/*
**
**     
**     Copyright 2018-2021 NXP
**     
**      SPDX-License-Identifier: BSD-3-Clause
**     
**     
*/

/*
 * Platform file with definitions for DDR type
 */


#ifndef __PLATFORM_DEF_H__
#define __PLATFORM_DEF_H__

#define NXP_DDRCLK_FREQ         1050000000

#define NUM_OF_DDRC         1
#define DDRC_NUM_DIMM       1
```

</details>


<details>
  <summary>Click to expand ddr1.h
</summary>

```c
/*
**
**     
**     Copyright 2018-2021 NXP
**     
**      SPDX-License-Identifier: BSD-3-Clause
**     
**     
*/

/*
 * Dummy file with definitions for DDR type
 */

#define NXP_DDR_ADDR            0x01080000
#define NXP_DDR_PHY1_ADDR       0x01400000
#define NXP_DDR2_ADDR           0x01090000
#define NXP_DDR_PHY2_ADDR       0x01600000

#define NXP_DCFG_ADDR       0x01EE0000
#define NXP_DDR2_ADDR       0x01090000
#define NXP_DDR_PHY2_ADDR   0x01600000
#define NXP_SYSCLK_FREQ     100000000
#define NXP_PLATFORM_CLK_DIVIDER    1
#define NXP_I2C_ADDR    0x02180000
#define NXP_SPD_EEPROM0     0x51
#define NXP_CCN_HN_F_0_ADDR     0x04200000
enum warm_boot {
    DDR_COLD_BOOT = 0,
    DDR_WARM_BOOT = 1,
    DDR_WRM_BOOT_NT_SUPPORTED = -1,
};

#ifndef DDRC_NUM_CS
#define DDRC_NUM_CS 4
#endif

typedef unsigned int uint16_t;

struct ddr_conf {
    int dimm_in_use[DDRC_NUM_DIMM];
    int cs_in_use;  /* bitmask, bit 0 for cs0, bit 1 for cs1, etc. */
    int cs_on_dimm[DDRC_NUM_DIMM];  /* bitmask */
    unsigned long long cs_base_addr[DDRC_NUM_CS];
    unsigned long long cs_size[DDRC_NUM_CS];
    unsigned long long base_addr;
    unsigned long long total_mem;
};

struct memctl_opt {
    int rdimm;
    unsigned int dbw_cap_shift;
    struct local_opts_s {
        unsigned int auto_precharge;
        unsigned int odt_rd_cfg;
        unsigned int odt_wr_cfg;
        unsigned int odt_rtt_norm;
        unsigned int odt_rtt_wr;
    } cs_odt[DDRC_NUM_CS];
    int ctlr_intlv;
    unsigned int ctlr_intlv_mode;
    unsigned int ba_intlv;
    int addr_hash;
    int ecc_mode;
    int ctlr_init_ecc;
    int self_refresh_in_sleep;
    int self_refresh_irq_en;
    int dynamic_power;
    /* memory data width 0 = 64-bit, 1 = 32-bit, 2 = 16-bit */
    unsigned int data_bus_dimm;
    unsigned int data_bus_used; /* on individual board */
    unsigned int burst_length;  /* BC4, OTF and BL8 */
    int otf_burst_chop_en;
    int mirrored_dimm;
    int quad_rank_present;
    int output_driver_impedance;
    int ap_en;
    int x4_en;

    int caslat_override;
    unsigned int caslat_override_value;
    int addt_lat_override;
    unsigned int addt_lat_override_value;

    unsigned int clk_adj;
    unsigned int cpo_sample;
    unsigned int wr_data_delay;

    unsigned int cswl_override;
    unsigned int wrlvl_override;
    unsigned int wrlvl_sample;
    unsigned int wrlvl_start;
    unsigned int wrlvl_ctl_2;
    unsigned int wrlvl_ctl_3;

    int half_strength_drive_en;
    int twot_en;
    int threet_en;
    unsigned int bstopre;
    unsigned int tfaw_ps;

    int rtt_override;
    unsigned int rtt_override_value;
    unsigned int rtt_wr_override_value;
    unsigned int rtt_park;

    int auto_self_refresh_en;
    unsigned int sr_it;
    unsigned int ddr_cdr1;
    unsigned int ddr_cdr2;

    unsigned int trwt_override;
    unsigned int trwt;
    unsigned int twrt;
    unsigned int trrt;
    unsigned int twwt;

    unsigned int vref_phy;
    unsigned int vref_dimm;
    unsigned int odt;
    unsigned int phy_tx_impedance;
    unsigned int phy_atx_impedance;
    unsigned int skip2d;
};

/* Parameters for a DDR dimm computed from the SPD */
struct dimm_params {
    /* DIMM organization parameters */
    char mpart[19];    /* guaranteed null terminated */

    unsigned int n_ranks;
    unsigned int die_density;
    unsigned long long rank_density;
    unsigned long long capacity;
    unsigned int primary_sdram_width;
    unsigned int ec_sdram_width;
    unsigned int rdimm;
    unsigned int package_3ds;   /* number of dies in 3DS */
    unsigned int device_width;  /* x4, x8, x16 components */
    unsigned int rc;

    /* SDRAM device parameters */
    unsigned int n_row_addr;
    unsigned int n_col_addr;
    unsigned int edc_config;    /* 0 = none, 1 = parity, 2 = ECC */
    unsigned int bank_addr_bits;
    unsigned int bank_group_bits;
    unsigned int burst_lengths_bitmask; /* BL=4 bit 2, BL=8 = bit 3 */

    /* mirrored DIMMs */
    unsigned int mirrored_dimm; /* only for ddr3 */

    /* DIMM timing parameters */

    int mtb_ps;      /* medium timebase ps */
    int ftb_10th_ps; /* fine timebase, in 1/10 ps */
    int taa_ps;      /* minimum CAS latency time */
    int tfaw_ps;     /* four active window delay */

    /*
     * SDRAM clock periods
     * The range for these are 1000-10000 so a short should be sufficient
     */
    int tckmin_x_ps;
    int tckmax_ps;

    /* SPD-defined CAS latencies */
    unsigned int caslat_x;

    /* basic timing parameters */
    int trcd_ps;
    int trp_ps;
    int tras_ps;

    int trfc1_ps;
    int trfc2_ps;
    int trfc4_ps;
    int trrds_ps;
    int trrdl_ps;
    int tccdl_ps;
    int trfc_slr_ps;

    int trc_ps; /* maximum = 254 ns + .75 ns = 254750 ps */
    int twr_ps; /* 15ns  for all speed bins */

    unsigned int refresh_rate_ps;
    unsigned int extended_op_srt;

    /* RDIMM */
    unsigned char rcw[16];  /* Register Control Word 0-15 */
    unsigned int dq_mapping[18];
    unsigned int dq_mapping_ors;
};

struct ddr_info {
    unsigned long clk;
    unsigned long long mem_base;
    unsigned int num_ctlrs;
    unsigned int dimm_on_ctlr;
    struct dimm_params dimm;
    struct memctl_opt opt;
    struct ddr_conf conf;
    struct ddr_cfg_regs ddr_reg;
    struct ccsr_ddr *ddr[NUM_OF_DDRC];
    uint16_t *phy[NUM_OF_DDRC];
    int *spd_addr;
    unsigned int ip_rev;
};
```

</details>

We'll omit `uboot_ddr1.c` because in our case DDR initialization occurs in BL2 (ATF).

Now we are interested in this fragment:
```c
// ...

static const struct rc_timing rcz[] = {
    {2100, 9, 8},
    {}
};

static const struct board_timing ram[] = {
    {0x02, rcz, 0x0709090D, 0x0D0F0F0C},
};
// ...
```

The values `0x0709090D` and `0x0D0F0F0C` should be the final values of the `DDR_WRLVL_CNTL_2` and `DDR_WRLVL_CNTL_3` registers respectively, however, [in the atf sources they are actually calculated](https://github.com/nxp-qoriq/atf/blob/bb4957067d4b96a6ee197a333425948e409e990d/drivers/nxp/ddr/nxp-ddr/ddr.c#L459), resulting in incorrect values, and then the BL31 bootloader failing to boot.

So, we need to recalculate these values:
```
0x0709090D = 8 * 0x01010101 + dimm[i].add1, so dimm[i].add1 should be 0xFF010105
0x0D0F0F0C = 8 * 0x01010101 + dimm[i].add2, so dimm[i].add2 should be 0x05070704
```

Also, the entry of new memory should be placed first to provide compatibility with the memory modules that are supported by default.
The final patch looks like this:

```diff
diff --git a/plat/nxp/soc-ls1046a/ls1046ardb/ddr_init.c b/plat/nxp/soc-ls1046a/ls1046ardb/ddr_init.c
index 6a9aa8357..343658900 100644
--- a/plat/nxp/soc-ls1046a/ls1046ardb/ddr_init.c
+++ b/plat/nxp/soc-ls1046a/ls1046ardb/ddr_init.c
@@ -199,9 +201,15 @@ static const struct rc_timing rce[] = {
    {}
 };
 
+static const struct rc_timing rcA[] = {
+   {2100, 9, 8}, // Our new memory
+   {}
+};
+
 static const struct board_timing udimm[] = {
-   {0x04, rce, 0x01020304, 0x06070805},
-   {0x1f, rce, 0x01020304, 0x06070805},
+   {0x02, rcA, 0xFF010105, 0x05070704}, // Our new memory
+   {0x04, rce, 0x01020304, 0x06070805}, // Micron MTA18ASF1G72AZ-2G3B1 8GB
+   {0x1f, rce, 0x01020304, 0x06070805}, // Micron MTA18ADF2G72AZ-2G6E1 16GB
 };
 
 int ddr_board_options(struct ddr_info *priv)

```

Rebuild the ATF in debug configuration, flash the eMMC (in our case), and the board successfully boots:

<details>
  <summary>Click to expand bootlog
</summary>

```sh
INFO:    SoC workaround for Errata A008850 Early-Phase was applied
INFO:    SoC workaround for Errata A010539 was applied
INFO:    RCW BOOT SRC is SD/EMMC
INFO:    SoC workaround for DDR Errata A008511 was applied
INFO:    SoC workaround for DDR Errata A009803 was applied
INFO:    SoC workaround for DDR Errata A009942 was applied
INFO:    SoC workaround for DDR Errata A010165 was applied
INFO:    platform clock 600000000
INFO:    DDR PLL1 2100000000
INFO:    DDR PLL2 500000000
INFO:    time base 8 ms
INFO:    Parse DIMM SPD(s)
INFO:    Controller 0
INFO:    DIMM 0
INFO:    addr 0x51
INFO:    checksum 0xbdd1733
INFO:    n_ranks 1
INFO:    rank_density 0x100000000
INFO:    capacity 0x100000000
INFO:    die density 0x5
INFO:    primary_sdram_width 64
INFO:    ec_sdram_width 0
INFO:    device_width 16
INFO:    package_3ds 0
INFO:    rdimm 0
INFO:    mirrored_dimm 0
INFO:    rc 0x2
INFO:    n_row_addr 16
INFO:    n_col_addr 10
INFO:    bank_addr_bits 0
INFO:    bank_group_bits 1
INFO:    edc_config 0
INFO:    burst_lengths_bitmask 0xc
INFO:    tckmin_x_ps 625
INFO:    tckmax_ps 1600
INFO:    caslat_x 0x7ffc00
INFO:    taa_ps 13750
INFO:    trcd_ps 13750
INFO:    trp_ps 13750
INFO:    tras_ps 32000
INFO:    trc_ps 45750
INFO:    trfc1_ps 350000
INFO:    trfc2_ps 260000
INFO:    trfc4_ps 160000
INFO:    tfaw_ps 30000
INFO:    trrds_ps 5300
INFO:    trrdl_ps 6400
INFO:    tccdl_ps 5000
INFO:    trfc_slr_ps 0
INFO:    twr_ps 15000
INFO:    refresh_rate_ps 7800000
INFO:    dq_mapping 0x16
INFO:    dq_mapping 0x36
INFO:    dq_mapping 0xc
INFO:    dq_mapping 0x35
INFO:    dq_mapping 0x16
INFO:    dq_mapping 0x36
INFO:    dq_mapping 0xc
INFO:    dq_mapping 0x35
INFO:    dq_mapping 0x0
INFO:    dq_mapping 0x0
INFO:    dq_mapping 0x16
INFO:    dq_mapping 0x36
INFO:    dq_mapping 0xc
INFO:    dq_mapping 0x35
INFO:    dq_mapping 0x16
INFO:    dq_mapping 0x36
INFO:    dq_mapping 0xc
INFO:    dq_mapping 0x35
INFO:    dq_mapping_ors 1
INFO:    done with controller 0
INFO:    cal cs
INFO:    cs_in_use = 1
INFO:    cs_on_dimm[0] = 1
NOTICE:  UDIMM 99U5713-015.A00G
INFO:    Time after parsing SPD 548 ms
INFO:    Synthesize configurations
INFO:    cs 0
INFO:         odt_rd_cfg 0x0
INFO:         odt_wr_cfg 0x4
INFO:         odt_rtt_norm 0x3
INFO:         odt_rtt_wr 0x0
INFO:         auto_precharge 0
INFO:    cs 1
INFO:         odt_rd_cfg 0x0
INFO:         odt_wr_cfg 0x0
INFO:         odt_rtt_norm 0x0
INFO:         odt_rtt_wr 0x0
INFO:         auto_precharge 0
INFO:    cs 2
INFO:         odt_rd_cfg 0x0
INFO:         odt_wr_cfg 0x0
INFO:         odt_rtt_norm 0x0
INFO:         odt_rtt_wr 0x0
INFO:         auto_precharge 0
INFO:    cs 3
INFO:         odt_rd_cfg 0x0
INFO:         odt_wr_cfg 0x0
INFO:         odt_rtt_norm 0x0
INFO:         odt_rtt_wr 0x0
INFO:         auto_precharge 0
INFO:    ctlr_init_ecc 0
INFO:    x4_en 0
INFO:    ap_en 0
INFO:    ctlr_intlv 0
INFO:    ctlr_intlv_mode 0
INFO:    ba_intlv 0x0
INFO:    data_bus_used 0
INFO:    otf_burst_chop_en 1
INFO:    burst_length 0x6
INFO:    dbw_cap_shift 0
INFO:    Assign binding addresses
INFO:    ctlr_intlv 0
INFO:    rank density 0x100000000
INFO:    CS 0
INFO:        base_addr 0x0
INFO:        size 0x100000000
INFO:    base 0x0
INFO:    Total mem by assignment is 0x100000000
INFO:    Calculate controller registers
INFO:    Skip CL mask for this speed 0x4000
INFO:    Skip caslat 0x4000
INFO:    cs_in_use = 0x1
INFO:    cs0
INFO:       _config = 0x80040412
INFO:    cs[0].bnds = 0xff
INFO:    sdram_cfg[0] = 0xc5000000
INFO:    sdram_cfg[1] = 0x401140
INFO:    sdram_cfg[2] = 0x0
INFO:    timing_cfg[0] = 0xd1770018
INFO:    timing_cfg[1] = 0xf2fc8265
INFO:    timing_cfg[2] = 0x5941a0
INFO:    timing_cfg[3] = 0x2161100
INFO:    timing_cfg[4] = 0x220002
INFO:    timing_cfg[5] = 0x5401400
INFO:    timing_cfg[6] = 0x0
INFO:    timing_cfg[7] = 0x26600000
INFO:    timing_cfg[8] = 0x5447a00
INFO:    timing_cfg[9] = 0x0
INFO:    dq_map[0] = 0x5b633558
INFO:    dq_map[1] = 0xd8cd56d8
INFO:    dq_map[2] = 0x3355b630
INFO:    dq_map[3] = 0xd4000001
INFO:    sdram_mode[0] = 0x3010631
INFO:    sdram_mode[1] = 0x100000
INFO:    sdram_mode[9] = 0x8400000
INFO:    sdram_mode[8] = 0x500
INFO:    sdram_mode[2] = 0x10631
INFO:    sdram_mode[3] = 0x100000
INFO:    sdram_mode[10] = 0x400
INFO:    sdram_mode[11] = 0x8400000
INFO:    sdram_mode[4] = 0x10631
INFO:    sdram_mode[5] = 0x100000
INFO:    sdram_mode[12] = 0x400
INFO:    sdram_mode[13] = 0x8400000
INFO:    sdram_mode[6] = 0x10631
INFO:    sdram_mode[7] = 0x100000
INFO:    sdram_mode[14] = 0x400
INFO:    sdram_mode[15] = 0x8400000
INFO:    interval = 0x1ffe0000
INFO:    zq_cntl = 0x8a090705
INFO:    ddr_sr_cntr = 0x0
INFO:    clk_cntl = 0x2400000
INFO:    cdr[0] = 0x80040000
INFO:    cdr[1] = 0xc1
INFO:    wrlvl_cntl[0] = 0x86750608
INFO:    wrlvl_cntl[1] = 0x709090d
INFO:    wrlvl_cntl[2] = 0xd0f0f0c
INFO:    debug[28] = 0x61
INFO:    Time before programming controller 798 ms
INFO:    Program controller registers
INFO:    Reading debug[9] as 0x39003900
INFO:    Reading debug[10] as 0x3d003c00
INFO:    Reading debug[11] as 0x45004500
INFO:    Reading debug[12] as 0x48004800
INFO:    cpo_min 0x39
INFO:    cpo_max 0x48
INFO:    debug[28] 0x6a0061
INFO:    Optimal cpo_sample 0x67
INFO:    *0x1080000 = 0xff
INFO:    *0x1080080 = 0x80040412
INFO:    *0x1080100 = 0x2161100
INFO:    *0x1080104 = 0xd1770018
INFO:    *0x1080108 = 0xf2fc8265
INFO:    *0x108010c = 0x5941a0
INFO:    *0x1080110 = 0xc5000000
INFO:    *0x1080114 = 0x401140
INFO:    *0x1080118 = 0x3010631
INFO:    *0x108011c = 0x100000
INFO:    *0x1080120 = 0x600086b
INFO:    *0x1080124 = 0x1ffe0000
INFO:    *0x1080128 = 0xdeadbeef
INFO:    *0x1080130 = 0x2400000
INFO:    *0x1080160 = 0x220002
INFO:    *0x1080164 = 0x5401400
INFO:    *0x108016c = 0x26600000
INFO:    *0x1080170 = 0x8a090705
INFO:    *0x1080174 = 0xc6750608
INFO:    *0x1080190 = 0x709090d
INFO:    *0x1080194 = 0xd0f0f0c
INFO:    *0x1080200 = 0x10631
INFO:    *0x1080204 = 0x100000
INFO:    *0x1080208 = 0x10631
INFO:    *0x108020c = 0x100000
INFO:    *0x1080210 = 0x10631
INFO:    *0x1080214 = 0x100000
INFO:    *0x1080220 = 0x500
INFO:    *0x1080224 = 0x8400000
INFO:    *0x1080228 = 0x400
INFO:    *0x108022c = 0x8400000
INFO:    *0x1080230 = 0x400
INFO:    *0x1080234 = 0x8400000
INFO:    *0x1080238 = 0x400
INFO:    *0x108023c = 0x8400000
INFO:    *0x1080250 = 0x5447a00
INFO:    *0x1080270 = 0xffff
INFO:    *0x1080280 = 0xdddeddde
INFO:    *0x1080284 = 0xdddedd21
INFO:    *0x1080288 = 0x22212221
INFO:    *0x108028c = 0x222122de
INFO:    *0x1080290 = 0x1
INFO:    *0x1080400 = 0x5b633558
INFO:    *0x1080404 = 0xd8cd56d8
INFO:    *0x1080408 = 0x3355b630
INFO:    *0x108040c = 0xd4000001
INFO:    *0x1080b20 = 0x8080
INFO:    *0x1080b24 = 0x80000000
INFO:    *0x1080b28 = 0x80040000
INFO:    *0x1080b2c = 0xc1
INFO:    *0x1080bf8 = 0x20502
INFO:    *0x1080bfc = 0x100
INFO:    *0x1080f04 = 0x2
INFO:    *0x1080f08 = 0xb
INFO:    *0x1080f0c = 0x14000c20
INFO:    *0x1080f24 = 0x39003900
INFO:    *0x1080f28 = 0x3d003c00
INFO:    *0x1080f2c = 0x45004500
INFO:    *0x1080f30 = 0x48004800
INFO:    *0x1080f34 = 0x7000
INFO:    *0x1080f48 = 0x1
INFO:    *0x1080f4c = 0x94000000
INFO:    *0x1080f50 = 0x11001000
INFO:    *0x1080f54 = 0x14001300
INFO:    *0x1080f58 = 0x1c001b00
INFO:    *0x1080f5c = 0x1f001f00
INFO:    *0x1080f60 = 0x18000000
INFO:    *0x1080f64 = 0x9000
INFO:    *0x1080f68 = 0x20
INFO:    *0x1080f70 = 0x6a0061
INFO:    *0x1080f94 = 0x80000000
INFO:    *0x1080f9c = 0x2c002e00
INFO:    *0x1080fa0 = 0x2e002e00
INFO:    *0x1080fa4 = 0x2d002d00
INFO:    *0x1080fa8 = 0x2b003100
INFO:    *0x1080fb0 = 0x3
INFO:    *0x1080fb4 = 0x1f1e1b1e
INFO:    *0x1080fb8 = 0x1d1d1e1c
INFO:    *0x1080fbc = 0x1f1f1f22
INFO:    *0x1080fc0 = 0x20201f1f
INFO:    *0x1080fc4 = 0x1f1e1e1e
INFO:    *0x1080fc8 = 0x1c1e1f1e
INFO:    *0x1080fcc = 0x1f1e1c20
INFO:    *0x1080fd0 = 0x1d1d1f1e
INFO:    *0x1080fd4 = 0x1f1e1b1e
INFO:    *0x1080fd8 = 0x1c1e1d1c
INFO:    *0x1080fdc = 0x1f1f1d20
INFO:    *0x1080fe0 = 0x1e201f1f
INFO:    *0x1080fe4 = 0x1f1d1b1e
INFO:    *0x1080fe8 = 0x1d1d1c1a
INFO:    *0x1080fec = 0x1f201f24
INFO:    *0x1080ff0 = 0x20202121
INFO:    *0x1080ff4 = 0x1f1f1f1f
INFO:    *0x1080ff8 = 0x1f1f1f1f
INFO:    *0x1080ffc = 0x1f000000

NOTICE:  4 GB DDR4, 64-bit, CL=15, ECC off
INFO:    Time used by DDR driver 1107 ms
INFO:    SoC workaround for Errata A008850 Post-Phase was applied
INFO:    RCW BOOT SRC is SD/EMMC
INFO:    esdhc_emmc_init
INFO:    Card detected successfully
INFO:    esdhc_wait_response: IRQSTAT CTOE set = 10001
INFO:    esdhc_wait_response: IRQSTAT CTOE set = 10001
INFO:    esdhc_wait_response: IRQSTAT CTOE set = 10001
INFO:    init done:
NOTICE:  BL2: v2.4(debug):lf-5.10.52-2.1.0-rc2-0-gbb4957067-dirty
NOTICE:  BL2: Built : 04:45:39, Sep  8 2021
INFO:    Configuring TrustZone Controller
INFO:    BL2: Doing platform setup
INFO:    BL2: Loading image id 3
INFO:    sd-mmc read done.
INFO:    sd-mmc read done.
INFO:    sd-mmc read done.
INFO:    Loading image id=3 at address 0xfbe00000
INFO:    sd-mmc read done.
INFO:    Image id=3 loaded: 0xfbe00000 - 0xfbe0a62d
INFO:    BL2: Loading image id 5
INFO:    sd-mmc read done.
INFO:    sd-mmc read done.
INFO:    sd-mmc read done.
INFO:    sd-mmc read done.
INFO:    sd-mmc read done.
INFO:    Loading image id=5 at address 0x82000000
INFO:    sd-mmc read done.
INFO:    Image id=5 loaded: 0x82000000 - 0x820c7dc6
NOTICE:  BL2: Booting BL31
INFO:    Entry point address = 0xfbe00000
INFO:    SPSR = 0x3cd
NOTICE:  BL31: v2.4(release):lf-5.10.52-2.1.0-rc2-0-gbb4957067
NOTICE:  BL31: Built : 04:45:39, Sep  8 2021
NOTICE:  Welcome to ls1046ardb BL31 Phase


U-Boot 2021.04+g1c0116f3da2+p0 (Sep 06 2021 - 08:48:23 +0000)

SoC:  LS1046AE Rev1.0 (0x87070010)
Clock Configuration:
       CPU0(A72):1800 MHz  CPU1(A72):1800 MHz  CPU2(A72):1800 MHz
       CPU3(A72):1800 MHz
       Bus:      600  MHz  DDR:      2100 MT/s  FMAN:     700  MHz
Reset Configuration Word (RCW):
       00000000: 0c150012 0e000000 00000000 00000000
       00000010: 11335559 40000012 60040000 c1000000
       00000020: 00000000 00000000 00000000 00238800
       00000030: 20124000 00003000 00000096 00000001
Model: LS1046A RDB Board
Board: LS1046ARDB, boot from SD
CPLD:  V2.3
PCBA:  V3.0
SERDES Reference Clocks:
SD1_CLK1 = 156.25MHZ, SD1_CLK2 = 100.00MHZ
DRAM:  3.9 GiB (DDR4, 64-bit, CL=15, ECC off)
Using SERDES1 Protocol: 4403 (0x1133)
Using SERDES2 Protocol: 21849 (0x5559)
NAND:  512 MiB
MMC:   FSL_SDHC: 0
Loading Environment from MMC... OK
EEPROM: NXID v1
In:    serial
Out:   serial
Err:   serial
SEC0:  RNG instantiated
Net:
MMC read: dev # 0, block # 18432, count 128 ...
Fman1: Uploading microcode version 106.4.18
eth0: fm1-mac3, eth1: fm1-mac4, eth2: fm1-mac5, eth3: fm1-mac6, eth4: fm1-mac9, eth5: fm1-mac10
Hit any key to stop autoboot:  0
=>
=>
=>
=>
=>
=>
=> boot
Trying to boot from mmc_a..
42144256 bytes read in 1790 ms (22.5 MiB/s)
31505 bytes read in 11 ms (2.7 MiB/s)
85155309 bytes read in 3607 ms (22.5 MiB/s)
## Loading init Ramdisk from Legacy Image at a0000000 ...
   Image Name:
   Image Type:   AArch64 Linux RAMDisk Image (gzip compressed)
   Data Size:    85155245 Bytes = 81.2 MiB
   Load Address: 00000000
   Entry Point:  00000000
   Verifying Checksum ... OK
## Flattened Device Tree blob at 90000000
   Booting using the fdt blob at 0x90000000
   Loading Ramdisk to 8aeca000, end 8ffffdad ... OK
   Loading Device Tree to 000000008aeaf000, end 000000008aec9b10 ... OK
PCIe1: pcie@3400000 Root Complex: no link
PCIe2: pcie@3500000 Root Complex: no link
PCIe3: pcie@3600000 Root Complex: no link
WARNING failed to get smmu node: FDT_ERR_NOTFOUND
WARNING failed to get smmu node: FDT_ERR_NOTFOUND

Starting kernel ...

[    0.000000] Booting Linux on physical CPU 0x0000000000 [0x410fd082]
[    0.000000] Linux version 5.10.52+ga11753a89ec6+p0 (oe-user@oe-host) (aarch64-poky-linux-gcc (GCC) 11.4.0, GNU ld (GNU Binutils) 2.38.20220708) #1 SMP PREEMPT Wed Sep 8 10:41:11 UTC 2021
[    0.000000] Machine model: LS1046A RDB Board
[    0.000000] earlycon: uart8250 at I/O port 0x0 (options '')
[    0.000000] Malformed early option 'earlycon'
[    0.000000] efi: UEFI not found.
[    0.000000] OF: reserved mem: initialized node qman-fqd, compatible id fsl,qman-fqd
[    0.000000] OF: reserved mem: initialized node qman-pfdr, compatible id fsl,qman-pfdr
[    0.000000] OF: reserved mem: initialized node bman-fbpr, compatible id fsl,bman-fbpr
[    0.000000] NUMA: No NUMA configuration found
[    0.000000] NUMA: Faking a node at [mem 0x0000000080000000-0x00000008ff7fffff]
[    0.000000] NUMA: NODE_DATA [mem 0x8ff014b00-0x8ff016fff]
[    0.000000] Zone ranges:
[    0.000000]   DMA      [mem 0x0000000080000000-0x00000000ffffffff]
[    0.000000]   DMA32    empty
[    0.000000]   Normal   [mem 0x0000000100000000-0x00000008ff7fffff]
[    0.000000] Movable zone start for each node
[    0.000000] Early memory node ranges
[    0.000000]   node   0: [mem 0x0000000080000000-0x00000000fbdfffff]
[    0.000000]   node   0: [mem 0x0000000880000000-0x00000008fbffffff]
[    0.000000]   node   0: [mem 0x00000008ff000000-0x00000008ff7fffff]
[    0.000000] Initmem setup node 0 [mem 0x0000000080000000-0x00000008ff7fffff]
[    0.000000] cma: Reserved 32 MiB at 0x00000000f9c00000
[    0.000000] psci: probing for conduit method from DT.
[    0.000000] psci: PSCIv1.1 detected in firmware.
[    0.000000] psci: Using standard PSCI v0.2 function IDs
[    0.000000] psci: MIGRATE_INFO_TYPE not supported.
[    0.000000] psci: SMC Calling Convention v1.2
[    0.000000] percpu: Embedded 24 pages/cpu s60120 r8192 d29992 u98304
[    0.000000] Detected PIPT I-cache on CPU0
[    0.000000] CPU features: detected: EL2 vector hardening
[    0.000000] CPU features: kernel page table isolation forced ON by KASLR
[    0.000000] CPU features: detected: Kernel page table isolation (KPTI)
[    0.000000] CPU features: detected: Spectre-v2
[    0.000000] CPU features: detected: ARM errata 1165522, 1319367, or 1530923
[    0.000000] Built 1 zonelists, mobility grouping on.  Total pages: 1001184
[    0.000000] Policy zone: Normal
[    0.000000] Kernel command line: console=ttyS0,115200 earlycon=uart8250 mtdparts=7e800000.flash:256m(nand_boot_a),193m(nand_boot_b),63m(nand_config) root=/dev/ram0 rw rootfstype=ramfs rdinit=/sbin/init
[    0.000000] Dentry cache hash table entries: 524288 (order: 10, 4194304 bytes, linear)
[    0.000000] Inode-cache hash table entries: 262144 (order: 9, 2097152 bytes, linear)
[    0.000000] mem auto-init: stack:off, heap alloc:off, heap free:off
[    0.000000] software IO TLB: mapped [mem 0x00000000f5c00000-0x00000000f9c00000] (64MB)
[    0.000000] Memory: 3765044K/4069376K available (21376K kernel code, 3076K rwdata, 9952K rodata, 6656K init, 1040K bss, 271564K reserved, 32768K cma-reserved)
[    0.000000] rcu: Preemptible hierarchical RCU implementation.
[    0.000000] rcu:     RCU event tracing is enabled.
[    0.000000] rcu:     RCU restricting CPUs from NR_CPUS=16 to nr_cpu_ids=4.
[    0.000000]  Trampoline variant of Tasks RCU enabled.
[    0.000000] rcu: RCU calculated value of scheduler-enlistment delay is 25 jiffies.
[    0.000000] rcu: Adjusting geometry for rcu_fanout_leaf=16, nr_cpu_ids=4
[    0.000000] NR_IRQS: 64, nr_irqs: 64, preallocated irqs: 0
[    0.000000] GIC: Adjusting CPU interface base to 0x000000000142f000
[    0.000000] GIC: Using split EOI/Deactivate mode
[    0.000000] random: get_random_bytes called from start_kernel+0x318/0x4e0 with crng_init=0
[    0.000000] arch_timer: cp15 timer(s) running at 25.00MHz (phys).
[    0.000000] clocksource: arch_sys_counter: mask: 0xffffffffffffff max_cycles: 0x5c40939b5, max_idle_ns: 440795202646 ns
[    0.000001] sched_clock: 56 bits at 25MHz, resolution 40ns, wraps every 4398046511100ns
[    0.000505] Console: colour dummy device 80x25
[    0.000547] Calibrating delay loop (skipped), value calculated using timer frequency.. 50.00 BogoMIPS (lpj=100000)
[    0.000554] pid_max: default: 32768 minimum: 301
[    0.000608] LSM: Security Framework initializing
[    0.000679] Mount-cache hash table entries: 8192 (order: 4, 65536 bytes, linear)
[    0.000704] Mountpoint-cache hash table entries: 8192 (order: 4, 65536 bytes, linear)
[    0.001414] rcu: Hierarchical SRCU implementation.
[    0.002359] EFI services will not be available.
[    0.002463] smp: Bringing up secondary CPUs ...
[    0.002739] Detected PIPT I-cache on CPU1
[    0.002765] CPU1: Booted secondary processor 0x0000000001 [0x410fd082]
[    0.003063] Detected PIPT I-cache on CPU2
[    0.003080] CPU2: Booted secondary processor 0x0000000002 [0x410fd082]
[    0.003363] Detected PIPT I-cache on CPU3
[    0.003379] CPU3: Booted secondary processor 0x0000000003 [0x410fd082]
[    0.003413] smp: Brought up 1 node, 4 CPUs
[    0.003423] SMP: Total of 4 processors activated.
[    0.003427] CPU features: detected: 32-bit EL0 Support
[    0.003430] CPU features: detected: CRC32 instructions
[    0.003434] CPU features: detected: 32-bit EL1 Support
[    0.013496] CPU: All CPU(s) started at EL2
[    0.013512] alternatives: patching kernel code
[    0.014095] devtmpfs: initialized
[    0.016398] KASLR enabled
[    0.016469] clocksource: jiffies: mask: 0xffffffff max_cycles: 0xffffffff, max_idle_ns: 7645041785100000 ns
[    0.016477] futex hash table entries: 1024 (order: 4, 65536 bytes, linear)
[    0.016847] pinctrl core: initialized pinctrl subsystem
[    0.017322] DMI not present or invalid.
[    0.017520] NET: Registered protocol family 16
[    0.018079] DMA: preallocated 512 KiB GFP_KERNEL pool for atomic allocations
[    0.018154] DMA: preallocated 512 KiB GFP_KERNEL|GFP_DMA pool for atomic allocations
[    0.018272] DMA: preallocated 512 KiB GFP_KERNEL|GFP_DMA32 pool for atomic allocations
[    0.018291] audit: initializing netlink subsys (disabled)
[    0.018356] audit: type=2000 audit(0.016:1): state=initialized audit_enabled=0 res=1
[    0.018784] thermal_sys: Registered thermal governor 'step_wise'
[    0.018787] thermal_sys: Registered thermal governor 'power_allocator'
[    0.019071] cpuidle: using governor menu
[    0.019139] Bman ver:0a02,02,01
[    0.021084] qman-fqd addr 0x00000008ff800000 size 0x800000
[    0.021087] qman-pfdr addr 0x00000008fc000000 size 0x2000000
[    0.021093] Qman ver:0a01,03,02,01
[    0.021177] hw-breakpoint: found 6 breakpoint and 4 watchpoint registers.
[    0.021214] ASID allocator initialised with 32768 entries
[    0.021804] Serial: AMBA PL011 UART driver
[    0.021848] imx mu driver is registered.
[    0.021866] imx rpmsg driver is registered.
[    0.041327] Machine: LS1046A RDB Board
[    0.041331] SoC family: QorIQ LS1046A
[    0.041334] SoC ID: svr:0x87070010, Revision: 1.0
[    0.053332] HugeTLB registered 1.00 GiB page size, pre-allocated 0 pages
[    0.053338] HugeTLB registered 32.0 MiB page size, pre-allocated 0 pages
[    0.053341] HugeTLB registered 2.00 MiB page size, pre-allocated 0 pages
[    0.053345] HugeTLB registered 64.0 KiB page size, pre-allocated 0 pages
[    0.054087] cryptd: max_cpu_qlen set to 1000
[    0.120045] raid6: neonx8   gen()  5125 MB/s
[    0.188072] raid6: neonx8   xor()  3689 MB/s
[    0.256104] raid6: neonx4   gen()  5258 MB/s
[    0.324137] raid6: neonx4   xor()  3785 MB/s
[    0.392166] raid6: neonx2   gen()  4642 MB/s
[    0.460197] raid6: neonx2   xor()  3542 MB/s
[    0.528231] raid6: neonx1   gen()  3590 MB/s
[    0.596256] raid6: neonx1   xor()  2878 MB/s
[    0.664283] raid6: int64x8  gen()  2741 MB/s
[    0.732311] raid6: int64x8  xor()  1638 MB/s
[    0.800336] raid6: int64x4  gen()  3063 MB/s
[    0.868380] raid6: int64x4  xor()  1725 MB/s
[    0.936398] raid6: int64x2  gen()  2874 MB/s
[    1.004426] raid6: int64x2  xor()  1534 MB/s
[    1.072455] raid6: int64x1  gen()  2200 MB/s
[    1.140486] raid6: int64x1  xor()  1143 MB/s
[    1.140489] raid6: using algorithm neonx4 gen() 5258 MB/s
[    1.140492] raid6: .... xor() 3785 MB/s, rmw enabled
[    1.140495] raid6: using neon recovery algorithm
[    1.140993] ACPI: Interpreter disabled.
[    1.143224] iommu: Default domain type: Passthrough
[    1.143359] vgaarb: loaded
[    1.143493] SCSI subsystem initialized
[    1.143641] usbcore: registered new interface driver usbfs
[    1.143658] usbcore: registered new interface driver hub
[    1.143673] usbcore: registered new device driver usb
[    1.144235] i2c i2c-0: IMX I2C adapter registered
[    1.144258] i2c i2c-0: using dma0chan16 (tx) and dma0chan17 (rx) for DMA transfers
[    1.144517] i2c i2c-1: IMX I2C adapter registered
[    1.145081] mc: Linux media interface: v0.10
[    1.145092] videodev: Linux video capture interface: v2.00
[    1.145122] pps_core: LinuxPPS API ver. 1 registered
[    1.145125] pps_core: Software ver. 5.3.6 - Copyright 2005-2007 Rodolfo Giometti <giometti@linux.it>
[    1.145132] PTP clock support registered
[    1.145231] EDAC MC: Ver: 3.0.0
[    1.145975] bman-fbpr addr 0x00000008fe000000 size 0x1000000
[    1.145998] Bman err interrupt handler present
[    1.146441] Bman portal initialised, cpu 0
[    1.146515] Bman portal initialised, cpu 1
[    1.146584] Bman portal initialised, cpu 2
[    1.146655] Bman portal initialised, cpu 3
[    1.146658] Bman portals initialised
[    1.147712] Qman err interrupt handler present
[    1.148062] QMan: Allocated lookup table at (____ptrval____), entry count 131073
[    1.148469] Qman portal initialised, cpu 0
[    1.148545] Qman portal initialised, cpu 1
[    1.148605] Qman portal initialised, cpu 2
[    1.148666] Qman portal initialised, cpu 3
[    1.148669] Qman portals initialised
[    1.148722] Bman: BPID allocator includes range 32:32
[    1.148754] Qman: FQID allocator includes range 256:256
[    1.148758] Qman: FQID allocator includes range 32768:32768
[    1.148792] Qman: CGRID allocator includes range 0:256
[    1.148929] Qman: pool channel allocator includes range 1025:15
[    1.148991] No USDPAA memory, no 'fsl,usdpaa-mem' in device-tree
[    1.149359] fsl-ifc 1530000.ifc: Freescale Integrated Flash Controller
[    1.149373] fsl-ifc 1530000.ifc: IFC version 1.4, 8 banks
[    1.149508] FPGA manager framework
[    1.149548] Advanced Linux Sound Architecture Driver Initialized.
[    1.150282] clocksource: Switched to clocksource arch_sys_counter
[    1.150377] VFS: Disk quotas dquot_6.6.0
[    1.150406] VFS: Dquot-cache hash table entries: 512 (order 0, 4096 bytes)
[    1.150499] pnp: PnP ACPI: disabled
[    1.153459] NET: Registered protocol family 2
[    1.153640] IP idents hash table entries: 65536 (order: 7, 524288 bytes, linear)
[    1.154634] tcp_listen_portaddr_hash hash table entries: 2048 (order: 3, 32768 bytes, linear)
[    1.154676] TCP established hash table entries: 32768 (order: 6, 262144 bytes, linear)
[    1.154794] TCP bind hash table entries: 32768 (order: 7, 524288 bytes, linear)
[    1.155020] TCP: Hash tables configured (established 32768 bind 32768)
[    1.155065] UDP hash table entries: 2048 (order: 4, 65536 bytes, linear)
[    1.155112] UDP-Lite hash table entries: 2048 (order: 4, 65536 bytes, linear)
[    1.155206] NET: Registered protocol family 1
[    1.155388] RPC: Registered named UNIX socket transport module.
[    1.155393] RPC: Registered udp transport module.
[    1.155395] RPC: Registered tcp transport module.
[    1.155398] RPC: Registered tcp NFSv4.1 backchannel transport module.
[    1.155691] PCI: CLS 0 bytes, default 64
[    1.155759] Trying to unpack rootfs image as initramfs...
[    3.227937] Freeing initrd memory: 83156K
[    3.228312] hw perfevents: enabled with armv8_cortex_a72 PMU driver, 7 counters available
[    3.228629] kvm [1]: IPA Size Limit: 44 bits
[    3.229304] kvm [1]: vgic interrupt IRQ9
[    3.229377] kvm [1]: Hyp mode initialized successfully
[    3.231108] Initialise system trusted keyrings
[    3.231173] workingset: timestamp_bits=42 max_order=20 bucket_order=0
[    3.231421] squashfs: version 4.0 (2009/01/31) Phillip Lougher
[    3.231541] NFS: Registering the id_resolver key type
[    3.231550] Key type id_resolver registered
[    3.231553] Key type id_legacy registered
[    3.231569] nfs4filelayout_init: NFSv4 File Layout Driver Registering...
[    3.231573] nfs4flexfilelayout_init: NFSv4 Flexfile Layout Driver Registering...
[    3.231585] jffs2: version 2.2. (NAND) © 2001-2006 Red Hat, Inc.
[    3.231691] fuse: init (API version 7.32)
[    3.231764] 9p: Installing v9fs 9p2000 file system support
[    3.253332] xor: measuring software checksum speed
[    3.254652]    8regs           :  7527 MB/sec
[    3.255789]    32regs          :  8679 MB/sec
[    3.257134]    arm64_neon      :  7331 MB/sec
[    3.257137] xor: using function: 32regs (8679 MB/sec)
[    3.257143] Key type asymmetric registered
[    3.257145] Asymmetric key parser 'x509' registered
[    3.257161] Block layer SCSI generic (bsg) driver version 0.4 loaded (major 243)
[    3.257215] io scheduler mq-deadline registered
[    3.257219] io scheduler kyber registered
[    3.268998] EINJ: ACPI disabled.
[    3.275963] Bus freq driver module loaded
[    3.281712] Serial: 8250/16550 driver, 4 ports, IRQ sharing enabled
[    3.283085] printk: console [ttyS0] disabled
[    3.283117] 21c0500.serial: ttyS0 at MMIO 0x21c0500 (irq = 48, base_baud = 18750000) is a 16550A
[    4.482591] printk: console [ttyS0] enabled
[    4.487018] 21c0600.serial: ttyS1 at MMIO 0x21c0600 (irq = 48, base_baud = 18750000) is a 16550A
[    4.496213] SuperH (H)SCI(F) driver initialized
[    4.501266] msm_serial: driver initialized
[    4.513335] brd: module loaded
[    4.519544] loop: module loaded
[    4.523517] megasas: 07.714.04.00-rc1
[    4.528094] imx ahci driver is registered.
[    4.532521] ahci-qoriq 3200000.sata: supply ahci not found, using dummy regulator
[    4.540060] ahci-qoriq 3200000.sata: supply phy not found, using dummy regulator
[    4.547483] ahci-qoriq 3200000.sata: supply target not found, using dummy regulator
[    4.555211] ahci-qoriq 3200000.sata: AHCI 0001.0301 32 slots 1 ports 6 Gbps 0x1 impl platform mode
[    4.564179] ahci-qoriq 3200000.sata: flags: 64bit ncq sntf pm clo only pmp fbs pio slum part ccc sds apst
[    4.574240] scsi host0: ahci-qoriq
[    4.577758] ata1: SATA max UDMA/133 mmio [mem 0x03200000-0x0320ffff] port 0x100 irq 59
[    4.586769] nand: device found, Manufacturer ID: 0x2c, Chip ID: 0xac
[    4.593127] nand: Micron MT29F4G08ABBFAH4
[    4.597135] nand: 512 MiB, SLC, erase size: 256 KiB, page size: 4096, OOB size: 256
[    4.605437] Bad block table found at page 131008, version 0x01
[    4.612479] Bad block table found at page 130944, version 0x01
[    4.618941] 3 cmdlinepart partitions found on MTD device 7e800000.flash
[    4.625560] Creating 3 MTD partitions on "7e800000.flash":
[    4.631049] 0x000000000000-0x000010000000 : "nand_boot_a"
[    4.638594] 0x000010000000-0x00001c100000 : "nand_boot_b"
[    4.646588] 0x00001c100000-0x000020000000 : "nand_config"
[    4.654567] fsl,ifc-nand 7e800000.nand: IFC NAND device at 0x7e800000, bank 0
[    4.663675] spi-nor spi1.0: s25fs512s (65536 Kbytes)
[    4.670700] spi-nor spi1.1: s25fs512s (65536 Kbytes)
[    4.680162] libphy: Fixed MDIO Bus: probed
[    4.685043] tun: Universal TUN/TAP device driver, 1.6
[    4.690725] thunder_xcv, ver 1.0
[    4.693960] thunder_bgx, ver 1.0
[    4.697200] nicpf, ver 1.0
[    4.700351] libphy: Freescale XGMAC MDIO Bus: probed
[    4.711831] libphy: Freescale XGMAC MDIO Bus: probed
[    4.725804] libphy: Freescale XGMAC MDIO Bus: probed
[    4.732060] libphy: Freescale XGMAC MDIO Bus: probed
[    4.738316] libphy: Freescale XGMAC MDIO Bus: probed
[    4.744572] libphy: Freescale XGMAC MDIO Bus: probed
[    4.750893] libphy: Freescale XGMAC MDIO Bus: probed
[    4.757228] libphy: Freescale XGMAC MDIO Bus: probed
[    4.763570] libphy: Freescale XGMAC MDIO Bus: probed
[    4.769807] libphy: Freescale XGMAC MDIO Bus: probed
[    4.785742] Freescale FM module, FMD API version 21.1.0
[    4.793306] Freescale FM Ports module
[    4.796970] fsl_mac: fsl_mac: FSL FMan MAC API based driver
[    4.802729] fsl_mac 1ae4000.ethernet: FMan MEMAC
[    4.807350] fsl_mac 1ae4000.ethernet: FMan MAC address: 00:04:9f:08:06:0b
[    4.814240] fsl_mac 1ae6000.ethernet: FMan MEMAC
[    4.818861] fsl_mac 1ae6000.ethernet: FMan MAC address: 00:04:9f:08:06:0c
[    4.825805] fsl_mac 1ae8000.ethernet: FMan MEMAC
[    4.830449] fsl_mac 1ae8000.ethernet: FMan MAC address: 00:04:9f:08:06:07
[    4.837389] fsl_mac 1aea000.ethernet: FMan MEMAC
[    4.842010] fsl_mac 1aea000.ethernet: FMan MAC address: 00:04:9f:08:06:08
[    4.848900] fsl_mac 1af0000.ethernet: FMan MEMAC
[    4.853521] fsl_mac 1af0000.ethernet: FMan MAC address: 00:04:9f:08:06:0a
[    4.861809] fsl_mac 1af2000.ethernet: FMan MEMAC
[    4.866436] fsl_mac 1af2000.ethernet: FMan MAC address: 00:04:9f:08:06:09
[    4.873262] fsl_dpa: FSL DPAA Ethernet driver
[    4.879080] fsl_dpa: fsl_dpa: Probed interface eth0
[    4.885316] fsl_dpa: fsl_dpa: Probed interface eth1
[    4.891677] fsl_dpa: fsl_dpa: Probed interface eth2
[    4.898152] fsl_dpa: fsl_dpa: Probed interface eth3
[    4.900398] ata1: SATA link down (SStatus 0 SControl 300)
[    4.904747] fsl_dpa: fsl_dpa: Probed interface eth4
[    4.915151] fsl_dpa: fsl_dpa: Probed interface eth5
[    4.920058] fsl_advanced: FSL DPAA Advanced drivers:
[    4.925022] fsl_proxy: FSL DPAA Proxy initialization driver
[    4.930754] fsl_oh: FSL FMan Offline Parsing port driver
[    4.936725] hclge is initializing
[    4.940066] hns3: Hisilicon Ethernet Network Driver for Hip08 Family - version
[    4.947295] hns3: Copyright (c) 2017 Huawei Corporation.
[    4.952631] e1000: Intel(R) PRO/1000 Network Driver
[    4.957510] e1000: Copyright (c) 1999-2006 Intel Corporation.
[    4.963272] e1000e: Intel(R) PRO/1000 Network Driver
[    4.968236] e1000e: Copyright(c) 1999 - 2015 Intel Corporation.
[    4.974172] igb: Intel(R) Gigabit Ethernet Network Driver
[    4.979571] igb: Copyright (c) 2007-2014 Intel Corporation.
[    4.985154] igbvf: Intel(R) Gigabit Virtual Function Network Driver
[    4.991426] igbvf: Copyright (c) 2009 - 2012 Intel Corporation.
[    4.997637] sky2: driver version 1.30
[    5.002282] usbcore: registered new interface driver r8152
[    5.007784] usbcore: registered new interface driver asix
[    5.013195] usbcore: registered new interface driver ax88179_178a
[    5.019566] VFIO - User Level meta-driver version: 0.3
[    5.027638] ehci_hcd: USB 2.0 'Enhanced' Host Controller (EHCI) Driver
[    5.034170] ehci-pci: EHCI PCI platform driver
[    5.038626] ehci-platform: EHCI generic platform driver
[    5.043964] ehci-orion: EHCI orion driver
[    5.048061] ehci-exynos: EHCI Exynos driver
[    5.052318] ohci_hcd: USB 1.1 'Open' Host Controller (OHCI) Driver
[    5.058504] ohci-pci: OHCI PCI platform driver
[    5.062961] ohci-platform: OHCI generic platform driver
[    5.068286] ohci-exynos: OHCI Exynos driver
[    5.072824] xhci-hcd xhci-hcd.0.auto: xHCI Host Controller
[    5.078318] xhci-hcd xhci-hcd.0.auto: new USB bus registered, assigned bus number 1
[    5.086038] xhci-hcd xhci-hcd.0.auto: hcc params 0x0220f66d hci version 0x100 quirks 0x0000000002010810
[    5.095451] xhci-hcd xhci-hcd.0.auto: irq 56, io mem 0x02f00000
[    5.101669] hub 1-0:1.0: USB hub found
[    5.105431] hub 1-0:1.0: 1 port detected
[    5.109477] xhci-hcd xhci-hcd.0.auto: xHCI Host Controller
[    5.114973] xhci-hcd xhci-hcd.0.auto: new USB bus registered, assigned bus number 2
[    5.122640] xhci-hcd xhci-hcd.0.auto: Host supports USB 3.0 SuperSpeed
[    5.129381] hub 2-0:1.0: USB hub found
[    5.133144] hub 2-0:1.0: 1 port detected
[    5.137229] xhci-hcd xhci-hcd.1.auto: xHCI Host Controller
[    5.142723] xhci-hcd xhci-hcd.1.auto: new USB bus registered, assigned bus number 3
[    5.150439] xhci-hcd xhci-hcd.1.auto: hcc params 0x0220f66d hci version 0x100 quirks 0x0000000002010810
[    5.159855] xhci-hcd xhci-hcd.1.auto: irq 58, io mem 0x03100000
[    5.166031] hub 3-0:1.0: USB hub found
[    5.169793] hub 3-0:1.0: 1 port detected
[    5.173824] xhci-hcd xhci-hcd.1.auto: xHCI Host Controller
[    5.179315] xhci-hcd xhci-hcd.1.auto: new USB bus registered, assigned bus number 4
[    5.186979] xhci-hcd xhci-hcd.1.auto: Host supports USB 3.0 SuperSpeed
[    5.193718] hub 4-0:1.0: USB hub found
[    5.197481] hub 4-0:1.0: 1 port detected
[    5.201754] usbcore: registered new interface driver uas
[    5.207099] usbcore: registered new interface driver usb-storage
[    5.215493] ftm-alarm 29d0000.timer: registered as rtc1
[    5.221515] i2c /dev entries driver
[    5.227115] ptp_qoriq: device tree node missing required elements, try automatic configuration
[    5.235794] pps pps0: new PPS source ptp0
[    5.265185] qoriq-cpufreq qoriq-cpufreq: Freescale QorIQ CPU frequency scaling driver
[    5.273627] sdhci: Secure Digital Host Controller Interface driver
[    5.279812] sdhci: Copyright(c) Pierre Ossman
[    5.284645] Synopsys Designware Multimedia Card Interface Driver
[    5.291565] sdhci-pltfm: SDHCI platform and OF driver helper
[    5.298835] ledtrig-cpu: registered to indicate activity on CPUs
[    5.305480] SMCCC: SOC_ID: ARCH_SOC_ID not implemented, skipping ....
[    5.312326] usbcore: registered new interface driver usbhid
[    5.317909] usbhid: USB HID core driver
[    5.322864] Freescale USDPAA process driver
[    5.323488] mmc0: SDHCI controller on 1560000.esdhc [1560000.esdhc] using ADMA 64-bit
[    5.327051] fsl-usdpaa: no region found
[    5.327055] Freescale USDPAA process IRQ driver
[    5.346573] optee: probing for conduit method.
[    5.351020] optee: api uid mismatch
[    5.354521] optee: probe of firmware:optee failed with error -22
[    5.365394] NET: Registered protocol family 26
[    5.369883] u32 classifier
[    5.372606]     input device check on
[    5.376267]     Actions configured
[    5.379959] Initializing XFRM netlink socket
[    5.384267] NET: Registered protocol family 10
[    5.389159] Segment Routing with IPv6
[    5.392865] sit: IPv6, IPv4 and MPLS over IPv4 tunneling driver
[    5.398991] NET: Registered protocol family 17
[    5.403444] NET: Registered protocol family 15
[    5.407909] Bridge firewalling registered
[    5.412009] 8021q: 802.1Q VLAN Support v1.8
[    5.416203] lib80211: common routines for IEEE802.11 drivers
[    5.418147] mmc0: new HS200 MMC card at address 0001
[    5.426865] 9pnet: Installing 9P2000 support
[    5.427124] mmcblk0: mmc0:0001 Q2J54A 3.64 GiB
[    5.431153] tsn generic netlink module v1 init...
[    5.435833] mmcblk0boot0: mmc0:0001 Q2J54A partition 1 2.00 MiB
[    5.440401] Key type dns_resolver registered
[    5.446454] mmcblk0boot1: mmc0:0001 Q2J54A partition 2 2.00 MiB
[    5.450697] registered taskstats version 1
[    5.456536] mmcblk0rpmb: mmc0:0001 Q2J54A partition 3 512 KiB, chardev (506:0)
[    5.460587] Loading compiled-in X.509 certificates
[    5.468803]  mmcblk0: p1 p2 p3
[    5.473488] Btrfs loaded, crc32c=crc32c-generic
[    5.482118] cfg80211: Loading compiled-in X.509 certificates for regulatory database
[    5.492907] cfg80211: Loaded X.509 cert 'sforshee: 00b28ddf47aef9cea7'
[    5.499514] platform regulatory.0: Direct firmware load for regulatory.db failed with error -2
[    5.502312] ALSA device list:
[    5.508142] platform regulatory.0: Falling back to sysfs fallback for: regulatory.db
[    5.511115]   No soundcards found.
[    5.524119] Freeing unused kernel memory: 6656K
[    5.554403] Run /sbin/init as init process
[    5.569167] systemd[1]: System time before build time, advancing clock.
[    5.577985] systemd[1]: systemd 250.5+ running in system mode (-PAM -AUDIT -SELINUX -APPARMOR +IMA -SMACK +SECCOMP -GCRYPT -GNUTLS -OPENSSL +ACL +BLKID -CURL -ELFUTILS -FIDO2 -IDN2 -IDN -IPTC +KMOD -LIBCRYPTSETUP +LIBFDISK -PCRE2 -PWQUALITY -P11KIT -QRENCODE -BZIP2 -LZ4 -XZ -ZLIB +ZSTD -BPF_FRAMEWORK +XKBCOMMON +UTMP +SYSVINIT default-hierarchy=hybrid)
[    5.609437] systemd[1]: Detected architecture arm64.

Welcome to Poky (Yocto Project Reference Distro) 4.0.18 (kirkstone)!

[    5.654569] systemd[1]: Hostname set to <ls1046ardb>.
[    5.659816] systemd[1]: Initializing machine ID from random generator.
[    5.682091] systemd[172]: /usr/lib/systemd/system-generators/systemd-gpt-auto-generator failed with exit status 1.
[    5.767930] systemd[1]: Queued start job for default target Multi-User System.
[    5.789838] systemd[1]: Created slice Slice /system/getty.
[  OK  ] Created slice Slice /system/getty.
[    5.811750] systemd[1]: Created slice Slice /system/modprobe.
[  OK  ] Created slice Slice /system/modprobe.
[    5.836063] systemd[1]: Created slice Slice /system/serial-getty.
[  OK  ] Created slice Slice /system/serial-getty.
[    5.859934] systemd[1]: Created slice User and Session Slice.
[  OK  ] Created slice User and Session Slice.
[    5.882654] systemd[1]: Started Dispatch Password Requests to Console Directory Watch.
[  OK  ] Started Dispatch Password …ts to Console Directory Watch.
[    5.906578] systemd[1]: Started Forward Password Requests to Wall Directory Watch.
[  OK  ] Started Forward Password R…uests to Wall Directory Watch.
[    5.930614] systemd[1]: Reached target Path Units.
[  OK  ] Reached target Path Units.
[    5.950423] systemd[1]: Reached target Remote File Systems.
[  OK  ] Reached target Remote File Systems.
[    5.970422] systemd[1]: Reached target Slice Units.
[  OK  ] Reached target Slice Units.
[    5.990432] systemd[1]: Reached target Swaps.
[  OK  ] Reached target Swaps.
[    6.010919] systemd[1]: Listening on RPCbind Server Activation Socket.
[  OK  ] Listening on RPCbind Server Activation Socket.
[    6.034413] systemd[1]: Reached target RPC Port Mapper.
[  OK  ] Reached target RPC Port Mapper.
[    6.054754] systemd[1]: Listening on Syslog Socket.
[  OK  ] Listening on Syslog Socket.
[    6.074611] systemd[1]: Listening on initctl Compatibility Named Pipe.
[  OK  ] Listening on initctl Compatibility Named Pipe.
[    6.098962] systemd[1]: Listening on Journal Audit Socket.
[  OK  ] Listening on Journal Audit Socket.
[    6.118742] systemd[1]: Listening on Journal Socket (/dev/log).
[  OK  ] Listening on Journal Socket (/dev/log).
[    6.142850] systemd[1]: Listening on Journal Socket.
[  OK  ] Listening on Journal Socket.
[    6.162932] systemd[1]: Listening on Network Service Netlink Socket.
[  OK  ] Listening on Network Service Netlink Socket.
[    6.186863] systemd[1]: Listening on udev Control Socket.
[  OK  ] Listening on udev Control Socket.
[    6.206720] systemd[1]: Listening on udev Kernel Socket.
[  OK  ] Listening on udev Kernel Socket.
[    6.226738] systemd[1]: Listening on User Database Manager Socket.
[  OK  ] Listening on User Database Manager Socket.
[    6.253766] systemd[1]: Mounting Huge Pages File System...
         Mounting Huge Pages File System...
[    6.277651] systemd[1]: Mounting POSIX Message Queue File System...
         Mounting POSIX Message Queue File System...
[    6.301849] systemd[1]: Mounting Kernel Debug File System...
         Mounting Kernel Debug File System...
[    6.322651] systemd[1]: Kernel Trace File System was skipped because of a failed condition check (ConditionPathExists=/sys/kernel/tracing).
[    6.337162] systemd[1]: Mounting Temporary Directory /tmp...
         Mounting Temporary Directory /tmp...
[    6.358674] systemd[1]: Create List of Static Device Nodes was skipped because of a failed condition check (ConditionFileNotEmpty=/lib/modules/5.10.52+ga11753a89ec6+p0/modules.devname).
[    6.376916] systemd[1]: Starting Load Kernel Module configfs...
         Starting Load Kernel Module configfs...
[    6.402423] systemd[1]: Starting Load Kernel Module drm...
         Starting Load Kernel Module drm...
[    6.425924] systemd[1]: Starting Load Kernel Module fuse...
         Starting Load Kernel Module fuse...
[    6.450086] systemd[1]: Starting RPC Bind...
         Starting RPC Bind...
[    6.466542] systemd[1]: File System Check on Root Device was skipped because of a failed condition check (ConditionPathIsReadWrite=!/).
[    6.479071] systemd[1]: systemd-journald.service: unit configures an IP firewall, but the local system does not support BPF/cgroup firewalling.
[    6.491954] systemd[1]: (This warning is only shown for the first unit using IP firewalling.)
[    6.502660] systemd[1]: Starting Journal Service...
         Starting Journal Service...
[    6.526694] systemd[1]: Load Kernel Modules was skipped because all trigger condition checks failed.
[    6.537396] systemd[1]: Mounting NFSD configuration filesystem...
         Mounting NFSD configuration filesystem...
[    6.564054] systemd[1]: Starting Generate network units from Kernel command line...
         Starting Generate network …ts from Kernel command line...
[    6.593209] systemd[1]: Starting Remount Root and Kernel File Systems...
         Starting Remount Root and Kernel File Systems...
[    6.622018] systemd[1]: Starting Apply Kernel Variables...
         Starting Apply Kernel Variables...
[    6.644383] systemd[1]: Starting Coldplug All udev Devices...
         Starting Coldplug All udev Devices...
[    6.668780] systemd[1]: Started RPC Bind.
[  OK  ] Started RPC Bind.
[    6.686620] systemd[1]: Started Journal Service.
[  OK  ] Started Journal Service.
[  OK  ] Mounted Huge Pages File System.
[  OK      6.722684] random: fast init done
0m] Mounted POSIX Message Queue File System.
[  OK  ] Mounted Kernel Debug File System.
[  OK  ] Mounted Temporary Directory /tmp.
[  OK  ] Finished Load Kernel Module configfs.
[  OK  ] Finished Load Kernel Module drm.
[  OK  ] Finished Load Kernel Module fuse.
[FAILED] Failed to mount NFSD configuration filesystem.
See 'systemctl status proc-fs-nfsd.mount' for details.
[DEPEND] Dependency failed for NFS server and services.
[DEPEND] Dependency failed for NFS Mount Daemon.
[  OK  ] Finished Generate network units from Kernel command line.
[  OK  ] Finished Remount Root and Kernel File Systems.
[  OK  ] Finished Apply Kernel Variables.
         Mounting FUSE Control File System...
         Mounting Kernel Configuration File System...
         Starting Flush Journal to Persistent Storage    6.989799] systemd-journald[190]: Received client request to flush runtime journal.
0m...
         Starting Create System Users...
[  OK  ] Mounted FUSE Control File System.
[  OK  ] Mounted Kernel Configuration File System.
[  OK  ] Finished Flush Journal to Persistent Storage.
[  OK  ] Finished Create System Users.
[  OK  ] Finished Coldplug All udev Devices.
         Starting Create Static Device Nodes in /dev...
[  OK  ] Finished Create Static Device Nodes in /dev.
[  OK  ] Reached target Preparation for Local File Systems.
         Mounting /var/volatile...
         Starting Rule-based Manage…for Device Events and Files...
[  OK  ] Mounted /var/volatile.
[  OK  ] Started Rule-based Manager for Device Events and Files.
         Starting Load/Save Random Seed...
[  OK  ] Reached target Local File Systems.
         Starting Rebuild Dynamic Linker Cache...
         Starting Create Volatile Files and Directories...
[  OK  ] Finished Rebuild Dynamic Linker Cache.
[  OK  ] Finished Create Volatile Files and Directories.
         Starting Rebuild Journal Catalog...
         Starting Network Time Synchronization...
         Starting Record System Boot/Shutdown in UTMP...
[  OK  ] Finished Rebuild Journal Catalog.
         Starting Update is Completed...
[  OK  ] Finished Update is Completed.
[  OK  ] Finished Record System Boot/Shutdown in UTMP.
[  OK  ] Started Network Time Synchronization.
[  OK  ] Reached target System Initialization.
[  OK  ] Started Daily Cleanup of Temporary Directories.
[  OK  ] Reached target System Time Set.
[  OK  ] Started Daily rotation of log files.
[  OK  ] Reached target Timer Units.
[  OK  ] Listening on D-Bus System Message Bus Socket.
         Starting sshd.socket...
[  OK  ] Listening on sshd.socket.
[  OK  ] Reached target Socket Units.
[  OK  ] Reached target Basic System.
[  OK  ] Reached target Hardware activated USB gadget.
[  OK  ] Started Job spooling tools.
[  OK  ] Started Periodic Command Scheduler.
         Starting D-Bus System Message [    7.817504] random: dbus-daemon: uninitialized urandom read (12 bytes read)
Bus...
[    7.828676] random: dbus-daemon: uninitialized urandom read (12 bytes read)
[  OK  ] Started Getty on tty1.
         Starting IPv6 Packet Filtering Framework...
         Starting IPv4 Packet Filtering Framework...
[  OK  ] Started Launcher Service.
[    7.897770] random: python3: uninitialized urandom read (24 bytes read)
[  OK  ] Started Serial Getty on ttyS0.
[  OK  ] Started Serial Getty on ttyS1.
[  OK  ] Reached target Login Prompts.
[  OK  ] Started System Logging Service.
         Starting User Login Management...
         Starting OpenSSH Key Generation...
[  OK  ] Started D-Bus System Message Bus.
[  OK  ] Finished IPv6 Packet Filtering Framework.
[  OK  ] Finished IPv4 Packet Filtering Framework.
[  OK  ] Reached target Preparation for Network.
         Starting Network Configuration...
[  OK  ] Started User Login Management.
[  OK  ] Started Network Configuration.
         Starting Network Name Resolution...
[  OK  ] Started Network Name Resolution.
[  OK  ] Reached target Network.
[  OK  ] Reached target Host and Network Name Lookups.
[  OK  ] Started NFS status monitor for NFSv2/3 locking..
[  OK  ] Started Respond to IPv6 Node Information Queries.
[  OK  ] Started Network Router Discovery Daemon.
[  OK  ] Reached target Multi-User System.
         Starting Record Runlevel Change in UTMP...
[  OK  ] Finished Record Runlevel Change in UTMP.
[  OK  ] Finished Load/Save Random Seed.
[  OK  ] Finished OpenSSH Key Generation.

Poky (Yocto Project Reference Distro) 4.0.18 ls1046ardb ttyS0

ls1046ardb login: root
Password:
root@ls1046ardb:~# free -h
               total        used        free      shared  buff/cache   available
Mem:           3.7Gi       133Mi       3.3Gi       9.0Mi       288Mi       3.2Gi
Swap:             0B          0B          0B
root@ls1046ardb:~#
```

</details>