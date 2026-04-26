# Hướng dẫn “tận răng”: Foundry + Sepolia (Ethereum testnet) + React (Vite) + Wagmi

> Mục tiêu: bạn deploy contract bằng **Foundry** lên **Sepolia**, sau đó trong **React** bạn sẽ:
>
> - **Connect wallet** (MetaMask)
> - **Đọc dữ liệu (read)** từ contract
> - **Ghi dữ liệu (write/tx)** lên contract + chờ tx confirm
> - Lấy **ETH balance** của ví
> - Lấy **event realtime** (watch) và **event history** (query logs)

Tài liệu này giả sử bạn mới biết cơ bản React và Foundry. Mình đi theo kiểu “làm được trước, hiểu dần”, giải thích từng thành phần được thêm vào.

---

## 0) Chuẩn bị

### 0.1. Bạn cần có

- **Node.js** (khuyến nghị LTS, ví dụ 20+)
- **Git**
- **MetaMask** cài trong browser
- **Sepolia ETH** (testnet) để trả phí gas
- 1 **RPC URL Sepolia** (Alchemy/Infura/QuickNode… đều được)

> Lưu ý bảo mật: **KHÔNG** commit private key lên Git. Chỉ để trong biến môi trường hoặc `.env.local` (đã gitignore).

### 0.2. Tạo 2 thư mục project (đề xuất)

Trong cùng 1 folder bạn có thể tách 2 project:

```
Wagmi/
  contracts/   # Foundry
  web/         # React
  docs/
```

Nếu bạn thích để riêng cũng được, nhưng để chung cho dễ copy ABI.

---

## 1) PHẦN A — Foundry: tạo contract + deploy lên Sepolia

### 1.1. Cài Foundry

Nếu bạn dùng **WSL** hoặc **Git Bash** (khuyến nghị trên Windows) thì chạy:

```bash
curl -L https://foundry.paradigm.xyz | bash
foundryup
```

Kiểm tra:

```bash
forge --version
cast --version
```

Nếu bạn không có `curl`, hãy cài Git for Windows (có Git Bash) hoặc dùng WSL.

### 1.2. Tạo project Foundry

Ở root workspace:

```bash
mkdir contracts
cd contracts
forge init
```

Giải thích nhanh:

- `forge init` tạo project Solidity chuẩn Foundry (có `src/`, `script/`, `test/`)

### 1.3. Viết contract mẫu: `Counter`

Tạo file `contracts/src/Counter.sol`:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

contract Counter {
    uint256 public number;

    event Increment(address indexed by, uint256 newNumber);
    event SetNumber(address indexed by, uint256 newNumber);

    constructor(uint256 initialNumber) {
        number = initialNumber;
    }

    function increment() external {
        number += 1;
        emit Increment(msg.sender, number);
    }

    function setNumber(uint256 newNumber) external {
        number = newNumber;
        emit SetNumber(msg.sender, number);
    }
}
```

Giải thích:

- `uint256 public number;` tạo **state variable** + tự sinh hàm getter `number()` (read)
- `increment()` và `setNumber()` là **write functions** (gửi tx)
- `event ...` giúp frontend nghe realtime và query history

### 1.4. Build contract

```bash
forge build
```

Kết quả quan trọng:

- ABI + bytecode sẽ nằm trong `contracts/out/Counter.sol/Counter.json`

### 1.5. Chuẩn bị biến môi trường để deploy

Bạn cần:

- `SEPOLIA_RPC_URL` (RPC provider)
- `PRIVATE_KEY` (private key của ví deploy)

#### Cách A (dễ nhất trong terminal bash): export biến môi trường

```bash
export SEPOLIA_RPC_URL="https://..."
export PRIVATE_KEY="0x..."
```

> `PRIVATE_KEY` là private key dạng hex, thường bắt đầu bằng `0x`.

### 1.6. Viết script deploy

Tạo file `contracts/script/DeployCounter.s.sol`:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "forge-std/Script.sol";
import "../src/Counter.sol";

contract DeployCounter is Script {
    function run() external returns (Counter deployed) {
        uint256 deployerPrivateKey = vm.envUint("PRIVATE_KEY");

        vm.startBroadcast(deployerPrivateKey);
        deployed = new Counter(0);
        vm.stopBroadcast();
    }
}
```

Giải thích từng phần:

- `forge-std/Script.sol`: thư viện chuẩn của Foundry cho scripting
- `vm.envUint("PRIVATE_KEY")`: đọc biến môi trường `PRIVATE_KEY` và parse sang uint
- `startBroadcast/stopBroadcast`: mọi tx giữa 2 lệnh này sẽ được ký và broadcast
- `new Counter(0)`: deploy contract với constructor param `initialNumber = 0`

### 1.7. Deploy lên Sepolia

Chạy:

```bash
forge script script/DeployCounter.s.sol:DeployCounter \
  --rpc-url "$SEPOLIA_RPC_URL" \
  --broadcast \
  -vvvv
```

Bạn sẽ thấy output có **Contract Address** (địa chỉ contract). Ghi lại:

- `COUNTER_ADDRESS=0x...`

> Tip: Ghi thêm block lúc deploy để query logs nhanh (tránh quét từ block 0).

### 1.8. Test nhanh bằng `cast` (optional nhưng rất hữu ích)

Giả sử:

- `COUNTER_ADDRESS=0x...`

Read `number()`:

```bash
cast call "$COUNTER_ADDRESS" "number()(uint256)" --rpc-url "$SEPOLIA_RPC_URL"
```

Gửi tx `increment()`:

```bash
cast send "$COUNTER_ADDRESS" "increment()" \
  --rpc-url "$SEPOLIA_RPC_URL" \
  --private-key "$PRIVATE_KEY"
```

---

## 2) PHẦN B — React + Wagmi: connect wallet + read/write + balance + events

### 2.1. Tạo React app bằng Vite

Từ root workspace:

```bash
npm create vite@latest web -- --template react
cd web
npm install
```

Giải thích:

- Vite là bundler nhanh, template `react` tạo project React **JavaScript** (đúng với bạn đang học)

> Nếu bạn muốn dùng TypeScript: thay `--template react` bằng `--template react-ts`. Khi đó các file sẽ là `.ts/.tsx` và có thêm type (không bắt buộc).

Chạy thử:

```bash
npm run dev
```

### 2.2. Cài thư viện Web3 cần thiết

Trong `web/`:

```bash
npm install wagmi viem @tanstack/react-query
```

Giải thích từng thư viện:

- `wagmi`: bộ hook React cho web3 (connect, read/write contract, watch events…)
- `viem`: engine EVM client của wagmi (RPC calls, logs, encoding…)
- `@tanstack/react-query`: wagmi dùng để cache + refetch dữ liệu “đúng kiểu React”

### 2.3. Thêm biến môi trường cho frontend

Tạo file `web/.env.local` (Vite yêu cầu prefix `VITE_`):

```env
VITE_SEPOLIA_RPC_URL=https://...
VITE_COUNTER_ADDRESS=0x...
VITE_COUNTER_DEPLOY_BLOCK=0
```

Giải thích:

- `VITE_SEPOLIA_RPC_URL`: RPC để frontend đọc chain (khuyến nghị dùng RPC riêng)
- `VITE_COUNTER_ADDRESS`: địa chỉ contract bạn vừa deploy
- `VITE_COUNTER_DEPLOY_BLOCK`: block number lúc deploy để query event history nhanh (điền số thật nếu có)

> Không commit `.env.local`.

### 2.4. Lấy ABI từ Foundry đưa qua frontend

Bạn có 2 cách:

#### Cách A (dễ hiểu): Copy ABI sang file JavaScript

Mở file `contracts/out/Counter.sol/Counter.json`, tìm key `abi` (mảng JSON), copy và tạo file:

`web/src/abi/counterAbi.js`

```js
export const counterAbi = [
  {
    type: "constructor",
    inputs: [{ name: "initialNumber", type: "uint256" }],
    stateMutability: "nonpayable",
  },
  {
    type: "event",
    name: "Increment",
    inputs: [
      { indexed: true, name: "by", type: "address" },
      { indexed: false, name: "newNumber", type: "uint256" },
    ],
  },
  {
    type: "event",
    name: "SetNumber",
    inputs: [
      { indexed: true, name: "by", type: "address" },
      { indexed: false, name: "newNumber", type: "uint256" },
    ],
  },
  {
    type: "function",
    name: "increment",
    inputs: [],
    outputs: [],
    stateMutability: "nonpayable",
  },
  {
    type: "function",
    name: "number",
    inputs: [],
    outputs: [{ name: "", type: "uint256" }],
    stateMutability: "view",
  },
  {
    type: "function",
    name: "setNumber",
    inputs: [{ name: "newNumber", type: "uint256" }],
    outputs: [],
    stateMutability: "nonpayable",
  },
];
```

> ABI trên là đúng với contract `Counter` ở phần Foundry. Nếu bạn sửa contract, hãy build lại và cập nhật ABI.

#### Cách B (ngắn gọn): Copy file JSON vào `src/abi/Counter.json`

Bạn có thể copy cả `Counter.json` hoặc chỉ `abi` sang JSON rồi import. Cách A dễ học hơn.

### 2.5. Setup Wagmi config

Tạo file `web/src/wagmi.js`:

```js
import { createConfig, http } from "wagmi";
import { sepolia } from "wagmi/chains";
import { metaMask } from "wagmi/connectors";

const sepoliaRpcUrl = import.meta.env.VITE_SEPOLIA_RPC_URL;

export const wagmiConfig = createConfig({
  chains: [sepolia],
  connectors: [metaMask()],
  transports: {
    [sepolia.id]: http(sepoliaRpcUrl),
  },
});
```

Giải thích:

- `chains: [sepolia]`: app chỉ làm việc với Sepolia (đơn giản, đỡ lỗi)
- `connectors: [metaMask()]`: cho người mới, chỉ cần MetaMask là đủ
- `transports`: wagmi/viem cần biết RPC để đọc chain và query logs

### 2.6. Bọc Provider ở entrypoint

Mở `web/src/main.jsx` và bọc thêm `WagmiProvider` + `QueryClientProvider`:

```jsx
import React from "react";
import ReactDOM from "react-dom/client";
import App from "./App";

import { WagmiProvider } from "wagmi";
import { QueryClient, QueryClientProvider } from "@tanstack/react-query";
import { wagmiConfig } from "./wagmi";

const queryClient = new QueryClient();

ReactDOM.createRoot(document.getElementById("root")).render(
  <React.StrictMode>
    <WagmiProvider config={wagmiConfig}>
      <QueryClientProvider client={queryClient}>
        <App />
      </QueryClientProvider>
    </WagmiProvider>
  </React.StrictMode>,
);
```

> Nếu bạn dùng TypeScript (`react-ts`), bạn có thể thấy cú pháp `document.getElementById("root")!` (non-null assertion). Với JavaScript (`react` template) thì **không dùng** dấu `!`.

Giải thích:

- `WagmiProvider`: đưa config xuống toàn bộ component tree để các hook wagmi dùng được
- `QueryClientProvider`: giúp wagmi cache/refetch dữ liệu (giống kiểu call API)

---

## 3) UI tối thiểu: Connect wallet + hiển thị chain + ETH balance

Thay `web/src/App.jsx` bằng phiên bản tối thiểu dưới đây.

### 3.1. App: connect/disconnect + switch Sepolia + ETH balance

`web/src/App.jsx`

```jsx
import { useMemo } from "react";
import {
  useAccount,
  useBalance,
  useConnect,
  useDisconnect,
  useSwitchChain,
  useChainId,
} from "wagmi";
import { sepolia } from "wagmi/chains";

import { CounterPanel } from "./CounterPanel";

export default function App() {
  const { address, isConnected } = useAccount();
  const chainId = useChainId();

  const {
    connect,
    connectors,
    isPending: isConnecting,
    error: connectError,
  } = useConnect();
  const { disconnect } = useDisconnect();
  const {
    switchChain,
    isPending: isSwitching,
    error: switchError,
  } = useSwitchChain();

  const { data: ethBalance, isLoading: isBalanceLoading } = useBalance({
    address,
    query: { enabled: Boolean(address) },
  });

  const isOnSepolia = chainId === sepolia.id;

  const metaMaskConnector = useMemo(
    () => connectors.find((c) => c.id === "metaMask") ?? connectors[0],
    [connectors],
  );

  return (
    <div
      style={{ maxWidth: 900, margin: "40px auto", fontFamily: "system-ui" }}
    >
      <h1>React + Wagmi + Sepolia</h1>

      <section
        style={{ padding: 16, border: "1px solid #ddd", borderRadius: 12 }}
      >
        <h2>1) Wallet</h2>

        {!isConnected ? (
          <button
            onClick={() => connect({ connector: metaMaskConnector })}
            disabled={isConnecting || !metaMaskConnector}
          >
            {isConnecting ? "Đang kết nối..." : "Connect MetaMask"}
          </button>
        ) : (
          <>
            <div>Address: {address}</div>
            <div>ChainId: {chainId}</div>

            {!isOnSepolia && (
              <button
                onClick={() => switchChain({ chainId: sepolia.id })}
                disabled={isSwitching}
              >
                {isSwitching ? "Đang switch..." : "Switch to Sepolia"}
              </button>
            )}

            <button onClick={() => disconnect()} style={{ marginLeft: 12 }}>
              Disconnect
            </button>

            <div style={{ marginTop: 12 }}>
              <h3>ETH Balance</h3>
              {isBalanceLoading ? (
                <div>Loading...</div>
              ) : (
                <div>
                  {ethBalance?.formatted} {ethBalance?.symbol}
                </div>
              )}
            </div>
          </>
        )}

        {connectError && (
          <p style={{ color: "crimson" }}>{connectError.message}</p>
        )}
        {switchError && (
          <p style={{ color: "crimson" }}>{switchError.message}</p>
        )}
      </section>

      <div style={{ height: 16 }} />

      <CounterPanel isWalletConnected={isConnected} isOnSepolia={isOnSepolia} />
    </div>
  );
}
```

Giải thích các hook wagmi bạn vừa dùng:

- `useAccount()`: lấy `address`, `isConnected`
- `useConnect()`: danh sách connector (MetaMask), hàm `connect()`
- `useDisconnect()`: `disconnect()`
- `useChainId()`: chain hiện tại của wallet
- `useSwitchChain()`: yêu cầu wallet switch sang Sepolia
- `useBalance()`: lấy ETH balance của address

---

## 4) Read/Write contract: Counter + chờ tx confirm + refetch

Tạo component `CounterPanel`.

### 4.1. `CounterPanel`: read number + increment + setNumber

Tạo file `web/src/CounterPanel.jsx`:

```jsx
import { useEffect, useMemo, useState } from "react";
import {
  useReadContract,
  useWriteContract,
  useWaitForTransactionReceipt,
  usePublicClient,
  useWatchContractEvent,
} from "wagmi";
import { sepolia } from "wagmi/chains";
import { parseAbiItem } from "viem";

import { counterAbi } from "./abi/counterAbi";

const counterAddress = import.meta.env.VITE_COUNTER_ADDRESS;
const deployBlock = BigInt(import.meta.env.VITE_COUNTER_DEPLOY_BLOCK || "0");

export function CounterPanel({ isWalletConnected, isOnSepolia }) {
  const [newNumber, setNewNumber] = useState("");
  const [history, setHistory] = useState([]);

  // Public client để query logs (event history)
  const publicClient = usePublicClient({ chainId: sepolia.id });

  // 1) READ: gọi view function number()
  const {
    data: currentNumber,
    isLoading: isNumberLoading,
    error: numberError,
    refetch: refetchNumber,
  } = useReadContract({
    address: counterAddress,
    abi: counterAbi,
    functionName: "number",
    chainId: sepolia.id,
    query: {
      enabled: Boolean(counterAddress),
    },
  });

  // 2) WRITE: gửi tx (increment / setNumber)
  const {
    writeContract,
    data: txHash,
    isPending: isWritePending,
    error: writeError,
  } = useWriteContract();

  // 3) WAIT: chờ tx confirm
  const {
    isLoading: isTxConfirming,
    isSuccess: isTxConfirmed,
    error: txError,
  } = useWaitForTransactionReceipt({
    hash: txHash,
    chainId: sepolia.id,
    query: { enabled: Boolean(txHash) },
  });

  // Khi tx confirmed thì refetch number()
  useEffect(() => {
    if (isTxConfirmed) {
      refetchNumber();
    }
  }, [isTxConfirmed, refetchNumber]);

  // 4) WATCH events realtime (logs mới)
  useWatchContractEvent({
    address: counterAddress,
    abi: counterAbi,
    eventName: "Increment",
    chainId: sepolia.id,
    onLogs: (logs) => {
      // logs có thể nhiều phần tử, ta append lên UI
      setHistory((prev) => [
        ...logs.map((l) => ({
          name: "Increment",
          by: String(l.args?.by ?? ""),
          newNumber: String(l.args?.newNumber ?? ""),
        })),
        ...prev,
      ]);
    },
  });

  useWatchContractEvent({
    address: counterAddress,
    abi: counterAbi,
    eventName: "SetNumber",
    chainId: sepolia.id,
    onLogs: (logs) => {
      setHistory((prev) => [
        ...logs.map((l) => ({
          name: "SetNumber",
          by: String(l.args?.by ?? ""),
          newNumber: String(l.args?.newNumber ?? ""),
        })),
        ...prev,
      ]);
    },
  });

  // 5) Query event history (logs cũ)
  const fetchHistory = async () => {
    if (!publicClient) return;

    const incrementEvent = parseAbiItem(
      "event Increment(address indexed by, uint256 newNumber)",
    );
    const setNumberEvent = parseAbiItem(
      "event SetNumber(address indexed by, uint256 newNumber)",
    );

    const [incLogs, setLogs] = await Promise.all([
      publicClient.getLogs({
        address: counterAddress,
        event: incrementEvent,
        fromBlock: deployBlock,
        toBlock: "latest",
      }),
      publicClient.getLogs({
        address: counterAddress,
        event: setNumberEvent,
        fromBlock: deployBlock,
        toBlock: "latest",
      }),
    ]);

    const all = [
      ...incLogs.map((l) => ({
        name: "Increment",
        by: String(l.args?.by ?? ""),
        newNumber: String(l.args?.newNumber ?? ""),
        blockNumber: l.blockNumber ?? 0n,
      })),
      ...setLogs.map((l) => ({
        name: "SetNumber",
        by: String(l.args?.by ?? ""),
        newNumber: String(l.args?.newNumber ?? ""),
        blockNumber: l.blockNumber ?? 0n,
      })),
    ]
      .sort((a, b) => Number(b.blockNumber - a.blockNumber))
      .map(({ name, by, newNumber }) => ({ name, by, newNumber }));

    setHistory(all);
  };

  const canInteract = isWalletConnected && isOnSepolia;

  const parsedNewNumber = useMemo(() => {
    if (newNumber.trim() === "") return null;
    const n = Number(newNumber);
    if (!Number.isFinite(n) || n < 0) return null;
    return BigInt(Math.floor(n));
  }, [newNumber]);

  return (
    <section
      style={{ padding: 16, border: "1px solid #ddd", borderRadius: 12 }}
    >
      <h2>2) Contract: Counter</h2>
      <div>Contract: {counterAddress}</div>

      <div style={{ marginTop: 12 }}>
        <h3>Read</h3>
        {isNumberLoading ? (
          <div>Loading number...</div>
        ) : (
          <div>number() = {String(currentNumber ?? "")}</div>
        )}
        {numberError && (
          <p style={{ color: "crimson" }}>{numberError.message}</p>
        )}

        <button onClick={() => refetchNumber()} style={{ marginTop: 8 }}>
          Refetch number()
        </button>
      </div>

      <div style={{ marginTop: 12 }}>
        <h3>Write</h3>

        <button
          disabled={!canInteract || isWritePending}
          onClick={() =>
            writeContract({
              address: counterAddress,
              abi: counterAbi,
              functionName: "increment",
              chainId: sepolia.id,
            })
          }
        >
          {isWritePending ? "Đang gửi tx..." : "increment()"}
        </button>

        <div style={{ marginTop: 8 }}>
          <input
            value={newNumber}
            onChange={(e) => setNewNumber(e.target.value)}
            placeholder="new number"
            style={{ padding: 8, width: 160 }}
          />
          <button
            style={{ marginLeft: 8 }}
            disabled={
              !canInteract || isWritePending || parsedNewNumber === null
            }
            onClick={() =>
              writeContract({
                address: counterAddress,
                abi: counterAbi,
                functionName: "setNumber",
                args: [parsedNewNumber],
                chainId: sepolia.id,
              })
            }
          >
            setNumber(n)
          </button>
        </div>

        <div style={{ marginTop: 8 }}>
          {txHash && <div>Tx hash: {txHash}</div>}
          {isTxConfirming && <div>Đang chờ confirm...</div>}
          {isTxConfirmed && <div>Tx confirmed ✅</div>}
        </div>

        {writeError && <p style={{ color: "crimson" }}>{writeError.message}</p>}
        {txError && <p style={{ color: "crimson" }}>{txError.message}</p>}

        {!isWalletConnected && <p>Hãy connect wallet để gửi tx.</p>}
        {isWalletConnected && !isOnSepolia && (
          <p>Hãy switch sang Sepolia để gửi tx.</p>
        )}
      </div>

      <div style={{ marginTop: 12 }}>
        <h3>Events</h3>

        <button onClick={fetchHistory} disabled={!publicClient}>
          Fetch event history
        </button>

        <div style={{ marginTop: 8 }}>
          {history.length === 0 ? (
            <div>(Chưa có event hoặc chưa fetch)</div>
          ) : (
            <ul>
              {history.slice(0, 20).map((e, idx) => (
                <li key={idx}>
                  <b>{e.name}</b> by {e.by} → newNumber = {e.newNumber}
                </li>
              ))}
            </ul>
          )}
        </div>
      </div>
    </section>
  );
}
```

Giải thích phần quan trọng:

- `useReadContract`: gọi hàm `view/pure` (không tốn gas)
- `useWriteContract`: tạo tx gọi hàm write (tốn gas)
- `useWaitForTransactionReceipt`: đợi tx lên block (confirmed)
- `useWatchContractEvent`: nghe log mới realtime
- `publicClient.getLogs`: query event history trong quá khứ

> `fromBlock`: nên dùng block deploy để query nhanh. Nếu để 0, query có thể lâu/timeout.

---

## 5) Chạy thử end-to-end

### 5.1. Start web

Trong `web/`:

```bash
npm run dev
```

### 5.2. Checklist nếu lỗi

- MetaMask đang ở **Sepolia** chưa?
- Bạn đã điền đúng `VITE_SEPOLIA_RPC_URL`, `VITE_COUNTER_ADDRESS` chưa?
- Contract ABI có khớp với contract đang deploy không?
- Ví có Sepolia ETH để gửi tx không?

---

## 6) Nâng cấp sau khi bạn chạy được (optional)

Nếu bạn muốn UX đẹp hơn (nhiều wallet, modal connect…), bạn có thể dùng:

- RainbowKit hoặc ConnectKit

Nhưng để học nền tảng, bản tối thiểu ở trên là tốt nhất.

---

## 7) Bạn muốn mình làm tiếp phần nào?

Bạn trả lời 1 câu thôi:

- Bạn đang dùng **wagmi version mấy** (hoặc paste `npm ls wagmi`), và bạn muốn mình hướng dẫn thêm:
  1. đọc ERC20 balance / allowance, hoặc
  2. listen event theo filter (by address), hoặc
  3. tổ chức ABI + address theo nhiều contract (multi-chain)?
