# radom
## 闭环速度控制系统流程图（含中断）

```mermaid
flowchart TD
    %% =========================
    %% 闭环速度控制系统（含中断）
    %% =========================

    A([上电/复位]) --> B[main(): 系统初始化]
    B --> B1[InitSysCtrl / InitGpio / InitPieCtrl / InitPieVectTable]
    B1 --> B2[InitPeripherals():\nADC_Init()\nInit_Cap1/2/3()\nEPWM_Enable(1)\nEPWM_Single_Init()\nEPWM_OUTPUT.PRD=EPWM_Configure(100)\nEPWM_OUTPUT.CMP=0.2*PRD]
    B2 --> B3[InitVariables():\n设置 bldcm.pole, hall.sysclk\npid_speed.Kp/Ki/Set/Outmax/Outmin\nbldcm_set.start=1, run=0]
    B3 --> B4[绑定中断向量:\nTINT0 -> CpuTimer0Isr\nECAP1/2/3 -> Cap_Isr]
    B4 --> B5[使能中断:\nPIEIER1(Timer0)\nPIEIER4(ECAP)\nIER|=M_INT1,M_INT4\nEINT]
    B5 --> C[ConfigCpuTimer(Timer0, 1ms)\nStartCpuTimer0()]
    C --> D{{for(;;) 主循环}}

    %% ---------- 启动流程 ----------
    D --> E{start==1 且 run==0 ?}
    E -- 是 --> E1[InitPeripherals() & InitVariables()\n(重新装载PID参数)]
    E1 --> E2[Motor_Ctrl() 启动一次驱动\n(给初始力矩)]
    E2 --> E3[run = 1]
    E -- 否 --> F

    %% ---------- 停止流程 ----------
    D --> S{stop==1 ?}
    S -- 是 --> S1[EPWM_Enable(0)\nstart=0; run=0; stop=0]
    S -- 否 --> F

    %% ---------- 1ms任务调度 ----------
    D --> F{cnt1 >= 1ms ?}
    F -- 是 --> G[Do_Every_1ms(): 闭环速度控制]
    F -- 否 --> H{cnt2 >= 500ms ?}
    H -- 是 --> H1[Do_Every_500ms(): LED_Run(); count++]
    H -- 否 --> D

    %% ---------- 1ms速度闭环 ----------
    G --> G1[测速:\nhall.cap_t1=Get_Cap1_Ts1()\nhall.cap_t2=Get_Cap1_Ts2()\nbldcm.speed=Motor_Speed()]
    G1 --> G2{run==1 ?}
    G2 -- 否 --> G4[仅采样/监视\n不更新CMP]
    G2 -- 是 --> G3[闭环调速:\npid_speed.Actual=bldcm.speed\nΔCMP=Pid_Ctrl(&pid_speed)\nEPWM_OUTPUT.CMP += ΔCMP\nCMP上限饱和(CMP<=PRD)]
    G3 --> G5[ADC采样:\nADC_Ctrl()\nudc/idc 读取]
    G4 --> G5
    G5 --> D

    %% =========================
    %% Timer0 中断：1ms 节拍
    %% =========================
    subgraph TIMER0_ISR[Timer0 ISR: CpuTimer0Isr 每1ms]
        T0[进入中断] --> T1[CntCtrl():\ncnt1++\ncnt2++]
        T1 --> T2[清中断标志\nPIEACK_GROUP1]
    end

    %% =========================
    %% ECAP 中断：换相驱动
    %% =========================
    subgraph ECAP_ISR[ECAP ISR: Cap_Isr 每次Hall边沿]
        C0[进入中断] --> C1[Motor_Ctrl():\n读取 hall.pos\n更新三相 EPWM_Set()]
        C1 --> C2[清ECAP标志\nPIEACK_GROUP4]
    end

    %% ---------- 中断与主循环关系 ----------
    TIMER0_ISR -. 提供节拍: cnt1/cnt2 .-> D
    ECAP_ISR -. 实时驱动换相 .-> D
