# radom
## 闭环速度控制系统流程图（含中断）

```mermaid
flowchart TD
  A[上电复位] --> B[main 初始化]
  B --> B1[InitPeripherals]
  B1 --> B2[InitVariables]
  B2 --> B3[绑定中断向量]
  B3 --> B4[使能中断 启动 Timer0]
  B4 --> C[进入主循环]

  C --> D{cnt1 到 1ms}
  D -- 否 --> C
  D -- 是 --> E[Do_Every_1ms]

  E --> E1[读取 cap_t1 cap_t2]
  E1 --> E2[Motor_Speed 得到 speed]
  E2 --> E3[pid Actual = speed]
  E3 --> E4[Pid_Ctrl 得到 delta CMP]
  E4 --> E5[CMP = CMP + delta]
  E5 --> E6[CMP 饱和 CMP <= PRD]
  E6 --> C

  T[Timer0 ISR 每 1ms] --> T1[cnt1++ cnt2++]
  T1 --> C

  H[Hall 边沿] --> I[ECAP ISR]
  I --> J[Motor_Ctrl 换相输出]