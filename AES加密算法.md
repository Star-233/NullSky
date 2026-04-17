AES 是一种**对称密钥加密算法**。所谓“对称”，意味着加密信息和解密信息使用的是**同一把钥匙**。

三种密钥长度：

| **算法名称**    | **密钥长度 (Key Size)** | **轮数 (Rounds)** | **安全级别**   |
| ----------- | ------------------- | --------------- | ---------- |
| **AES-128** | 128 bit             | 10 轮            | 极高（商业通用）   |
| **AES-192** | 192 bit             | 12 轮            | 极高         |
| **AES-256** | 256 bit             | 14 轮            | 顶级（军方/政府级） |
## 加密过程

```mermaid
graph TD
    subgraph "输入阶段"
        P["明文 (128-bit)"]
        K["原始密钥 (128/192/256-bit)"]
    end

    K --> KE["密钥扩展 (Key Expansion)<br>生成多组轮密钥"]
    P --> ARK0["初始轮密钥加 (AddRoundKey)"]

    subgraph "加密循环 (以 AES-128 为例)"
        ARK0 --> R1["第 1 轮 (标准轮)"]
        R1 --> R2["..."]
        R2 --> R9["第 9 轮 (标准轮)"]
    end

    R9 --> FR["最终轮 (无列混淆 MixColumns)"]
    FR --> C["密文 (128-bit)"]

    KE -.-> ARK0
    KE -.-> R1
    KE -.-> R9
    KE -.-> FR
```

### 单个标准轮的内部操作

```mermaid
graph TD
    %% 数据初始化阶段
    subgraph Init ["<b>数据准备 (Data Preparation)</b>"]
        direction TB
        Raw["输入字符串<br>(如: 'Hello AES!')"] --> ToBytes["转换为字节流<br>(16字节为一组)"]
        ToBytes --> Matrix["<b>填充状态矩阵 (State)</b><br>按列填充到 4x4 矩阵<br>s0,0 s0,1 s0,2 s0,3<br>s1,0 s1,1 s1,2 s1,3<br>s2,0 s2,1 s2,2 s2,3<br>s3,0 s3,1 s3,2 s3,3"]
    end

    Matrix --> SubBytes

    %% 第一步：字节代换
    subgraph SubBytes ["<b>Step 1: SubBytes (字节代换)</b>"]
        direction TB
        SB_In["当前 4x4 状态矩阵"] --> SB_Map["查表映射 (S-Box)"]
        SB_Map --> SB_Logic["将每个字节替换为<br>另一个对应的字节"]
        SB_Logic --> SB_Out["代换后的矩阵"]
    end

    SubBytes --> ShiftRows

    %% 第二步：行移位
    subgraph ShiftRows ["<b>Step 2: ShiftRows (行移位)</b>"]
        direction TB
        SR_In["矩阵按行操作"]
        SR_In --> R0["第 0 行: 不动"]
        SR_In --> R1["第 1 行: 循环左移 1 字节"]
        SR_In --> R2["第 2 行: 循环左移 2 字节"]
        SR_In --> R3["第 3 行: 循环左移 3 字节"]
        R0 & R1 & R2 & R3 --> SR_Out["打乱行顺序后的矩阵"]
    end

    ShiftRows --> MixColumns

    %% 第三步：列混淆
    subgraph MixColumns ["<b>Step 3: MixColumns (列混淆)</b>"]
        direction TB
        MC_In["矩阵按列操作"] --> MC_Math["列向量乘以<br>一个固定的数学矩阵"]
        MC_Math --> MC_Logic["使得一列中的 4 个字节<br>完全混合在一起"]
        MC_Logic --> MC_Out["混淆后的矩阵"]
    end

    MixColumns --> AddRoundKey

    %% 第四步：轮密钥加
    subgraph AddRoundKey ["<b>Step 4: AddRoundKey (轮密钥加)</b>"]
        direction TB
        ARK_S["当前状态矩阵"] --> ARK_XOR["逐位异或计算 (XOR)"]
        ARK_K["本轮子密钥 (128位)"] --> ARK_XOR
        ARK_XOR --> ARK_Out["本轮加密完成"]
    end

    AddRoundKey --> Out(("输出结果<br>进入下一轮加密"))

    %% 样式美化
    style Init fill:#f5f5f5,stroke:#333
    style SubBytes fill:#f9f,stroke:#333
    style ShiftRows fill:#bbf,stroke:#333
    style MixColumns fill:#bfb,stroke:#333
    style AddRoundKey fill:#fbb,stroke:#333
```

## 解密过程
```mermaid
graph TD
    subgraph "输入阶段"
        C["密文 (128-bit)"]
        K["原始密钥 (128/192/256-bit)"]
    end

    K --> KE["密钥扩展 (Key Expansion)<br>生成多组轮密钥"]
    C --> ARK10["初始轮密钥加 (AddRoundKey)<br>使用最后一轮子密钥"]

    subgraph "解密循环 (以 AES-128 为例)"
        ARK10 --> R1["第 1 轮 (逆向标准轮)"]
        R1 --> R2["..."]
        R2 --> R9["第 9 轮 (逆向标准轮)"]
    end

    R9 --> FR["最终轮 (无逆列混淆 InvMixColumns)"]
    FR --> P["明文 (128-bit)"]

    KE -.-> ARK10["使用 Round Key 10"]
    KE -.-> R1["使用 Round Key 9"]
    KE -.-> R9["使用 Round Key 1"]
    KE -.-> FR["使用 Round Key 0 (原始密钥)"]
```

```mermaid
graph TD
    %% 数据初始化阶段
    subgraph Init ["<b>解密准备 (Decryption Prep)</b>"]
        direction TB
        Cipher["输入密文<br>(16字节块)"] --> Matrix["<b>填充状态矩阵 (State)</b><br>按列填充到 4x4 矩阵<br>c0,0 c0,1 c0,2 c0,3<br>c1,0 c1,1 c1,2 c1,3<br>c2,0 c2,1 c2,2 c2,3<br>c3,0 c3,1 c3,2 c3,3"]
    end

    Matrix --> InvShiftRows

    %% 第一步：逆向行移位
    subgraph InvShiftRows ["<b>Step 1: InvShiftRows (逆行移位)</b>"]
        direction TB
        ISR_In["矩阵按行操作"]
        ISR_In --> R0["第 0 行: 不动"]
        ISR_In --> R1["第 1 行: 循环右移 1 字节"]
        ISR_In --> R2["第 2 行: 循环右移 2 字节"]
        ISR_In --> R3["第 3 行: 循环右移 3 字节"]
        R0 & R1 & R2 & R3 --> ISR_Out["行位置还原后的矩阵"]
    end

    InvShiftRows --> InvSubBytes

    %% 第二步：逆向字节代换
    subgraph InvSubBytes ["<b>Step 2: InvSubBytes (逆字节代换)</b>"]
        direction TB
        ISB_In["当前状态矩阵"] --> ISB_Map["逆向查表映射 (Inverse S-Box)"]
        ISB_Map --> ISB_Logic["通过逆 S 盒还原<br>每个字节的原始值"]
        ISB_Logic --> ISB_Out["还原后的矩阵"]
    end

    InvSubBytes --> AddRoundKey

    %% 第三步：轮密钥加
    subgraph AddRoundKey ["<b>Step 3: AddRoundKey (轮密钥加)</b>"]
        direction TB
        ARK_S["当前状态矩阵"] --> ARK_XOR["逐位异或计算 (XOR)"]
        ARK_K["本轮对应的子密钥<br>(倒序匹配)"] --> ARK_XOR
        ARK_XOR --> ARK_Out["密钥混淆还原"]
    end

    AddRoundKey --> InvMixColumns

    %% 第四步：逆向列混淆
    subgraph InvMixColumns ["<b>Step 4: InvMixColumns (逆列混淆)</b>"]
        direction TB
        IMC_In["矩阵按列操作"] --> IMC_Math["列向量乘以<br>逆向固定矩阵"]
        IMC_Math --> IMC_Logic["通过更复杂的数学运算<br>解除列内部的混合"]
        IMC_Logic --> IMC_Out["列关系还原完成"]
    end

    InvMixColumns --> Out(("输出结果<br>进入下一轮解密"))

    %% 样式美化
    style Init fill:#f5f5f5,stroke:#333
    style InvSubBytes fill:#f9f,stroke:#333
    style InvShiftRows fill:#bbf,stroke:#333
    style InvMixColumns fill:#bfb,stroke:#333
    style AddRoundKey fill:#fbb,stroke:#333
```

解密过程和加密过程不是直接颠倒的，不过这是数学问题了。

## 工作模式
由于 AES 一次只能加密 128bit，所以出现了不同的工作模式解决这一问题。

### ECB 工作模式
ECB 是最基础的工作模式，他直接独立地处理每一个 128bit 数据。

加密过程：
```mermaid
graph TD
    subgraph Block_1 ["<b>数据块 1</b>"]
        P1["明文块 1<br>(128-bit)"] --> E1["AES 加密引擎<br>(使用密钥 K)"]
        E1 --> C1["密文块 1<br>(128-bit)"]
    end

    subgraph Block_2 ["<b>数据块 2</b>"]
        P2["明文块 2<br>(128-bit)"] --> E2["AES 加密引擎<br>(使用密钥 K)"]
        E2 --> C2["密文块 2<br>(128-bit)"]
    end

    subgraph Block_N ["<b>数据块 N</b>"]
        PN["明文块 N<br>(128-bit)"] --> EN["AES 加密引擎<br>(使用密钥 K)"]
        EN --> CN["密文块 N<br>(128-bit)"]
    end

    style E1 fill:#f9f,stroke:#333
    style E2 fill:#f9f,stroke:#333
    style EN fill:#f9f,stroke:#333
```

解密过程：
```mermaid
graph TD
    subgraph D_Block_1 ["<b>密文还原 1</b>"]
        C1["密文块 1<br>(128-bit)"] --> DE1["AES 解密引擎<br>(使用密钥 K)"]
        DE1 --> DP1["明文块 1<br>(还原)"]
    end

    subgraph D_Block_2 ["<b>密文还原 2</b>"]
        C2["密文块 2<br>(128-bit)"] --> DE2["AES 解密引擎<br>(使用密钥 K)"]
        DE2 --> DP2["明文块 2<br>(还原)"]
    end

    style DE1 fill:#bbf,stroke:#333
    style DE2 fill:#bbf,stroke:#333
```

### CBC 工作模式
与 ECB 模式不同，**CBC (Cipher Block Chaining，密码块链接)** 模式引入了一个“链式”结构。每一块明文在加密之前，必须先与前一个块的**密文**进行异或运算。

加密过程：
```mermaid
graph TD
    IV["<b>IV (初始化向量)</b><br>随机生成"] --> XOR1["⊕ XOR (异或)"]
    P1["明文块 1<br>(128-bit)"] --> XOR1
    
    XOR1 --> E1["AES 加密引擎<br>(Key K)"]
    E1 --> C1["<b>密文块 1</b>"]

    C1 --> XOR2["⊕ XOR (异或)"]
    P2["明文块 2<br>(128-bit)"] --> XOR2
    
    XOR2 --> E2["AES 加密引擎<br>(Key K)"]
    E2 --> C2["<b>密文块 2</b>"]

    C2 --> Next["... 传递至下一块"]

    style E1 fill:#f9f,stroke:#333
    style E2 fill:#f9f,stroke:#333
    style IV fill:#fff4dd,stroke:#d4a017
```

解密过程：
```mermaid
graph TD
    C1["<b>密文块 1</b>"] --> D1["AES 解密引擎<br>(Key K)"]
    IV["<b>IV (初始化向量)</b>"] --> XOR1["⊕ XOR (异或)"]
    D1 --> XOR1
    XOR1 --> P1["明文块 1<br>(还原)"]

    C2["<b>密文块 2</b>"] --> D2["AES 解密引擎<br>(Key K)"]
    C1 --> XOR2["⊕ XOR (异或)"]
    D2 --> XOR2
    XOR2 --> P2["明文块 2<br>(还原)"]

    style D1 fill:#bbf,stroke:#333
    style D2 fill:#bbf,stroke:#333
    style IV fill:#fff4dd,stroke:#d4a017
```


### CTR 工作模式
**CTR (Counter, 计数器模式)** 是目前最受欢迎的 AES 工作模式之一。它的设计理念非常独特：它不直接加密你的数据，而是加密一个**“计数器”**，将其变成一串随机乱码，再通过异或（XOR）运算把乱码“盖”在你的数据上。

这种设计让它从“块加密”变成了“**流**加密”，具备了极高的性能。

**Nonce** 是 **"Number used once"**（只使用一次的数字）的缩写。
加密过程：
```mermaid
graph TD
    subgraph Block_1 ["<b>数据块 1</b>"]
        N1["Nonce + 计数器 001"] --> E1["AES 加密引擎<br>(Key K)"]
        E1 --> MSK1["密钥流块 1<br>(乱码)"]
        P1["明文块 1"] --> XOR1["⊕ XOR (异或)"]
        MSK1 --> XOR1
        XOR1 --> C1["<b>密文块 1</b>"]
    end

    subgraph Block_2 ["<b>数据块 2</b>"]
        N2["Nonce + 计数器 002"] --> E2["AES 加密引擎<br>(Key K)"]
        E2 --> MSK2["密钥流块 2<br>(乱码)"]
        P2["明文块 2"] --> XOR2["⊕ XOR (异或)"]
        MSK2 --> XOR2
        XOR2 --> C2["<b>密文块 2</b>"]
    end

    subgraph Block_N ["<b>数据块 N</b>"]
        NN["Nonce + 计数器 N"] --> EN["AES 加密引擎<br>(Key K)"]
        EN --> MSKN["密钥流块 N<br>(乱码)"]
        PN["明文块 N"] --> XORN["⊕ XOR (异或)"]
        MSKN --> XORN
        XORN --> CN["<b>密文块 N</b>"]
    end

    style E1 fill:#f9f,stroke:#333
    style E2 fill:#f9f,stroke:#333
    style EN fill:#f9f,stroke:#333
    style N1 fill:#fff4dd,stroke:#d4a017
    style N2 fill:#fff4dd,stroke:#d4a017
```


解密过程：
```mermaid
graph TD
    subgraph D_Block_1 ["<b>解密块 1</b>"]
        N1["Nonce + 计数器 001"] --> E1["AES 加密引擎<br>(Key K)"]
        E1 --> MSK1["密钥流块 1"]
        C1["密文块 1"] --> XOR1["⊕ XOR (异或)"]
        MSK1 --> XOR1
        XOR1 --> P1["明文块 1 (还原)"]
    end

    subgraph D_Block_2 ["<b>解密块 2</b>"]
        N2["Nonce + 计数器 002"] --> E2["AES 加密引擎<br>(Key K)"]
        E2 --> MSK2["密钥流块 2"]
        C2["密文块 2"] --> XOR2["⊕ XOR (异或)"]
        MSK2 --> XOR2
        XOR2 --> P2["明文块 2 (还原)"]
    end

    style E1 fill:#f9f,stroke:#333
    style E2 fill:#f9f,stroke:#333
```
#### 块加密是怎么变成流加密的？
之前的加密是直接对明文进行加密，所以就需要明文为 128bit。不过，现在 AES是对 $\text{Nonce}+\text{计数器数}$ 加密，然后再对明文进行异或，所以只需要 $\text{Nonce}+\text{计数器数}$ 满足 128bit 的要求就好了。又因为异或是可以单个位进行运算的，自然变成了流加密。

## 填充模式
由于AES需要128bit的明文，那么不足时需要填充。

需要注意的是，并不是所有工作模式都需要填充

| **模式**        | **是否需要 Padding** | **原因**                                                    |
| ------------- | ---------------- | --------------------------------------------------------- |
| **ECB / CBC** | **是**            | 它们是直接对明文块进行操作，必须对齐 16 字节。                                 |
| **CTR / GCM** | **否**            | 它们把 AES 变成了“流加密”。由于它是用乱码流和明文做异或，明文有多少字节就异或多少字节，多余的乱码直接丢弃。 |

### PKCS#7 填充
