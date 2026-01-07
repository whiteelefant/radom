# radom
## 闭环速度控制系统流程图（含中断）

```mermaid
flowchart TD
    A[上电复位] --> B[main 初始化]
    B --> B1[初始化外设 InitPeripherals]
    B1 --> B2[初始化变量 InitVariables]
    B2 --> B3[绑定中断向量]
    B3 --> B4[使能中断并启动 Timer0 1ms]
    B4 --> C[进入主循环 for(;;)]

    %% --- 主循环调度 ---
    C --> D{cnt1 到 1ms}
    D -- 否 --> C
    D -- 是 --> E[执行 Do_Every_1ms 速度闭环]

    %% --- 1ms 速度闭环 ---
    E --> E1[读取捕获值 cap_t1 cap_t2]
    E1 --> E2[计算速度 Motor_Speed 得到 bldcm_speed]
    E2 --> E3[pid_actual = bldcm_speed]
    E3 --> E4[delta_cmp = Pid_Ctrl]
    E4 --> E5[cmp = cmp + delta_cmp]
    E5 --> E6[cmp 饱和限制 cmp <= prd]
    E6 --> C

    %% --- Timer0 中断：提供节拍 ---
    T[Timer0 ISR 每 1ms] --> T1[cnt1++ cnt2++]
    T1 --> C

    %% --- ECAP 中断：换相驱动（不展开内部） ---
    H[Hall 边沿] --> I[ECAP ISR Cap_Isr]
    I --> J[Motor_Ctrl 换相输出]
