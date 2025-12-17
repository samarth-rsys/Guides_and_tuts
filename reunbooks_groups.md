
## PPP_BBL Group Allocation Pattern

### **Groups are NOT test-type specific** - they are **CI/CD Pipeline Resource Groups** for parallel execution!

| Group | Pipeline Resource | Test Type Usage | Purpose |
|-------|-------------------|-----------------|---------|
| **PPP_BBL_0** | `ppp-bbl-group-0` | QoS SRL tests | Generic tests |
| **PPP_BBL_1** | `ppp-bbl-group-1` | - | Generic tests |
| **PPP_BBL_2** | `ppp-bbl-group-2` | - | Generic tests |
| **PPP_BBL_3** | `ppp-bbl-group-3` | ADF replace (single-stage) | **13 runbooks** from `TC_04_000_071_pppoe_Single_Stage_ADF_CoA_replaced.yml` |
| **PPP_BBL_4** | `ppp-bbl-group-4` | ADF replace (single-stage) | **13 runbooks** (2nd batch) from same template |
| **PPP_BBL_5** | `ppp-bbl-group-5` | ADF multistage upgrade/deactivate | **10 runbooks** from `TC_04_000_072_pppoe_Multistage_ADF_upgrade.yml` and `TC_04_000_073_pppoe_Multistage_ADF_CoA_deactivate.yml` |
| **PPP_BBL_7** | `ppp-bbl-group-7` | QoS Accounting, CoA profile changes | QoS runbooks |

### **Key Insight**

The groups allow **parallel test execution** in CI/CD pipelines:
- Each group has its own **physical interface** (access + network)
- Each group has its own **VLAN range** 
- Each group has a unique **MAC modifier** (to avoid MAC conflicts)
- Tests using the **same group cannot run in parallel** (resource contention)
- Tests using **different groups can run in parallel**

### **For BLR1LEAF01 specifically:**

All groups (3, 4, 5) currently share the same physical interface `ifp-0/1/11` and access interface `eno2.1230`, differentiated only by:
- **OUTER_VLAN** (e.g., 1146, 1148, 1150, etc.)
- **MAC_MODIFIER** (3, 4, 5)
- **REMOTE_ID** (DALAB.A4.BLANK_BBL_3, BBL_4, BBL_5)

### **Which group to use?**

| Test Type | Recommended Group |
|-----------|-------------------|
| **Single-stage ADF replace** (CoA replaces one filter) | PPP_BBL_3 or PPP_BBL_4 |
| **Multistage ADF** (CoA adds multiple filters progressively) | PPP_BBL_5 |
| **QoS/Accounting tests** | PPP_BBL_7 |
| **General/Basic PPPoE tests** | PPP_BBL_0, PPP_BBL_1, PPP_BBL_2 |

So for your pppoe_QoS_CoA_2p_3p_SRL_increase.yml - using **PPP_BBL_5** is fine since it's a CoA test, though traditionally it might use PPP_BBL_0 (QoS SRL tests). The key is to ensure the **OUTER_VLAN** matches what's configured for that group in your testbed.
