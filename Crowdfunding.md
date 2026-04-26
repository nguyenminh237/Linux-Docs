# Wagmi “tận răng” cho project này: `foundry-test/` (Crowdfunding) + `web3-interface/` (React)

> Bạn đang có sẵn 2 phần trong repo:
>
> - **Smart contract**: `foundry-test/src/Crowdfunding.sol` (đã có test + script deploy)
> - **Frontend**: `web3-interface/` (React + TypeScript + Tailwind) — hiện đang có `ethers` nhưng phần connect/interaction chưa làm xong
>
> Mục tiêu của guide này:
>
> 1. Dùng **Wagmi** để connect MetaMask
> 2. **Read** từ contract (contract balance, số funders, mapping `s_funderToAmount`, `owner`…)
> 3. **Write** lên contract (payable `fund()`, `withdraw()` onlyOwner)
> 4. **Events**: nghe realtime `Funded/Withdrawn` và query event history
>
> Reference docs: https://wagmi.sh/react/getting-started

---

## 0) Tổng quan contract `Crowdfunding` (để hiểu frontend sẽ gọi gì)

Contract ở `foundry-test/src/Crowdfunding.sol` có:

- **Write**
  - `fund()` **payable**: gửi ETH vào contract, require số USD >= `MINIMUM_USD` (5 USD)
  - `withdraw()` onlyOwner: rút toàn bộ ETH về `owner()`
- **Read**
  - `getFundersLength()`
  - `s_funderToAmount(address)` (mapping: 1 ví fund bao nhiêu ETH)
  - `s_isFunders(address)` (mapping: có phải funder không)
  - `owner()`
  - `MINIMUM_USD()`
- **Events**
  - `Funded(address funder, uint256 value)`
  - `Withdrawn(uint256 value)`

### 0.1. Vì sao `fund()` hay bị revert?

Trong `fund()` có:

```solidity
require(i_ethUsdPriceFeed.getConversionRate(msg.value) >= MINIMUM_USD, "no available amount");
```

Nghĩa là nếu bạn gửi quá ít ETH thì tx sẽ revert với message **`no available amount`**.

---

## 1) Foundry: chạy test + deploy Sepolia (lấy đúng địa chỉ contract cho frontend)

### 1.1. Chạy unit test (để hiểu flow đúng)

Trong `foundry-test/`:

```bash
forge test -vv
```

Những test quan trọng (để frontend “match” behavior):

- `test_can_fund`: sau `fund()` thì contract balance tăng, mapping tăng, funders length tăng, emit event `Funded`
- `test_revert_fund`: gửi thiếu ETH thì revert `no available amount`
- `test_can_withdraw`: chỉ owner rút được, emit `Withdrawn`

### 1.2. Deploy lên Sepolia

Deploy script: `foundry-test/script/DeployCrowdfunding.s.sol`

```bash
cd foundry-test
forge script script/DeployCrowdfunding.s.sol:DeployCrowdfunding \
  --rpc-url "$SEPOLIA_RPC_URL" \
  --private-key "$PRIVATE_KEY" \
  --broadcast \
  -vvvv
```

### 1.3. Lấy contract address + deploy block từ broadcast (cực quan trọng cho “event history”)

Sau khi deploy, Foundry lưu log tại:

- `foundry-test/broadcast/DeployCrowdfunding.s.sol/11155111/run-latest.json`

Trong repo hiện tại, giá trị đang là:

- `Crowdfunding` address: `0xAaf16d3da4A0c09194e2cf0e1ee1F2F2ed8F2759`
- Deploy block: `0xa3c661` (hex) = `10733153` (decimal)

> Bạn sẽ dùng 2 thông tin này cho frontend để:
>
> - read/write đúng contract
> - query event history nhanh (fromBlock = deployBlock)

---

## 2) Frontend: cài Wagmi (trên project `web3-interface/` hiện có)

> `web3-interface/` đang là **React + TypeScript + Vite**.
> Bạn học React “JavaScript” vẫn dùng được y hệt — TypeScript chỉ thêm type, không đổi logic.

### 2.1. Cài dependencies của Wagmi

Trong `web3-interface/`:

```bash
npm install wagmi viem @tanstack/react-query
```

Giải thích:

- `wagmi`: hook connect wallet, read/write contract, watch events…
- `viem`: EVM client (encode/decode ABI, parse logs, format Ether…)
- `@tanstack/react-query`: wagmi dùng để cache/refetch data theo kiểu React

### 2.2. (Optional) Nếu bạn muốn bỏ ethers/web3modal

Hiện `web3-interface/package.json` có `ethers` và `@web3modal/ethers`. Nếu bạn **chỉ dùng wagmi**, có thể uninstall:

```bash
npm uninstall ethers @web3modal/ethers
```

Không bắt buộc. Nhưng bỏ đi sẽ “đỡ rối” cho người mới.

---

## 3) Frontend: cấu hình biến môi trường

Tạo file `web3-interface/.env.local`:

```env
VITE_SEPOLIA_RPC_URL=https://YOUR_SEPOLIA_RPC
VITE_CROWDFUNDING_ADDRESS=0xAaf16d3da4A0c09194e2cf0e1ee1F2F2ed8F2759
VITE_CROWDFUNDING_DEPLOY_BLOCK=10733153
```

Giải thích:

- `VITE_SEPOLIA_RPC_URL`: RPC Sepolia (Alchemy/Infura/QuickNode…)
- `VITE_CROWDFUNDING_ADDRESS`: địa chỉ contract Sepolia
- `VITE_CROWDFUNDING_DEPLOY_BLOCK`: block deploy (để query logs nhanh)

> Vite chỉ expose env var bắt đầu bằng `VITE_`.

---

## 4) Frontend: ABI của `Crowdfunding` (lấy từ Foundry)

### 4.1. Lấy ABI từ file build của Foundry

ABI “chuẩn” nằm ở:

- `foundry-test/out/Crowdfunding.sol/Crowdfunding.json` (key `abi`)

### 4.2. Tạo file ABI cho frontend

Tạo file `web3-interface/src/contracts/crowdfundingAbi.ts` và copy **mảng `abi`** (không cần bytecode).

Ví dụ (đã đúng với contract hiện tại của bạn):

```ts
export const crowdfundingAbi = [
  {
    type: "constructor",
    inputs: [
      { name: "ethUsdPriceFeed", type: "address", internalType: "address" },
    ],
    stateMutability: "nonpayable",
  },
  { type: "fallback", stateMutability: "payable" },
  { type: "receive", stateMutability: "payable" },

  {
    type: "function",
    name: "fund",
    inputs: [],
    outputs: [],
    stateMutability: "payable",
  },
  {
    type: "function",
    name: "withdraw",
    inputs: [],
    outputs: [],
    stateMutability: "nonpayable",
  },

  {
    type: "function",
    name: "getFundersLength",
    inputs: [],
    outputs: [{ name: "", type: "uint256", internalType: "uint256" }],
    stateMutability: "view",
  },
  {
    type: "function",
    name: "s_funderToAmount",
    inputs: [{ name: "", type: "address", internalType: "address" }],
    outputs: [{ name: "", type: "uint256", internalType: "uint256" }],
    stateMutability: "view",
  },
  {
    type: "function",
    name: "s_isFunders",
    inputs: [{ name: "", type: "address", internalType: "address" }],
    outputs: [{ name: "", type: "bool", internalType: "bool" }],
    stateMutability: "view",
  },
  {
    type: "function",
    name: "owner",
    inputs: [],
    outputs: [{ name: "", type: "address", internalType: "address" }],
    stateMutability: "view",
  },

  {
    type: "event",
    name: "Funded",
    anonymous: false,
    inputs: [
      {
        name: "funder",
        type: "address",
        indexed: true,
        internalType: "address",
      },
      {
        name: "value",
        type: "uint256",
        indexed: false,
        internalType: "uint256",
      },
    ],
  },
  {
    type: "event",
    name: "Withdrawn",
    anonymous: false,
    inputs: [
      {
        name: "value",
        type: "uint256",
        indexed: false,
        internalType: "uint256",
      },
    ],
  },
] as const;
```

Giải thích:

- ABI là “bản mô tả interface” để wagmi/viem encode function call + decode dữ liệu trả về/logs.
- `as const` giúp TypeScript infer đúng tên hàm/event.

---

## 5) Frontend: tạo wagmi config

Tạo file `web3-interface/src/wagmi.ts`:

```ts
import { createConfig, http } from "wagmi";
import { sepolia } from "wagmi/chains";
import { metaMask } from "wagmi/connectors";

const sepoliaRpcUrl = import.meta.env.VITE_SEPOLIA_RPC_URL as string;

export const wagmiConfig = createConfig({
  chains: [sepolia],
  connectors: [metaMask()],
  transports: {
    [sepolia.id]: http(sepoliaRpcUrl),
  },
});
```

Giải thích từng phần:

- `chains: [sepolia]`: app chỉ làm việc trên Sepolia
- `connectors: [metaMask()]`: dùng MetaMask extension
- `transports`: chỉ định RPC cho chain (để read chain + query logs)

---

## 6) Frontend: bọc Provider (Wagmi + React Query)

Sửa `web3-interface/src/main.tsx`:

```tsx
import { StrictMode } from "react";
import { createRoot } from "react-dom/client";
import "./index.css";
import App from "./App";

import { WagmiProvider } from "wagmi";
import { QueryClient, QueryClientProvider } from "@tanstack/react-query";
import { wagmiConfig } from "./wagmi";

const queryClient = new QueryClient();

createRoot(document.getElementById("root")!).render(
  <StrictMode>
    <WagmiProvider config={wagmiConfig}>
      <QueryClientProvider client={queryClient}>
        <App />
      </QueryClientProvider>
    </WagmiProvider>
  </StrictMode>,
);
```

Giải thích:

- `WagmiProvider` đưa config xuống toàn bộ component tree
- `QueryClientProvider` giúp caching/refetch data như call REST API

---

## 7) Frontend: khai báo address + deploy block (theo env)

Sửa `web3-interface/src/contracts/contractData.ts` thành:

```ts
export const contractAddr = import.meta.env
  .VITE_CROWDFUNDING_ADDRESS as `0x${string}`;
export const deployBlock = BigInt(
  import.meta.env.VITE_CROWDFUNDING_DEPLOY_BLOCK || "0",
);
```

Giải thích:

- `contractAddr` kiểu `0x${string}` giúp viem hiểu đây là address
- `deployBlock` dùng `BigInt` vì viem dùng `bigint` cho block number

---

## 8) Connect wallet MetaMask + ETH balance

Trong `web3-interface/src/App.tsx`, bạn sẽ dùng các hook:

- `useAccount()` lấy `address`, `isConnected`
- `useConnect()` để connect MetaMask
- `useDisconnect()` để disconnect
- `useChainId()` + `useSwitchChain()` để đảm bảo đang ở Sepolia
- `useBalance()` để lấy ETH balance

Ví dụ skeleton connect:

```tsx
import { useMemo } from "react";
import { sepolia } from "wagmi/chains";
import {
  useAccount,
  useBalance,
  useChainId,
  useConnect,
  useDisconnect,
  useSwitchChain,
} from "wagmi";

export function WalletSection() {
  const { address, isConnected } = useAccount();
  const chainId = useChainId();

  const { connect, connectors, isPending: isConnecting, error } = useConnect();
  const { disconnect } = useDisconnect();
  const { switchChain, isPending: isSwitching } = useSwitchChain();

  const isOnSepolia = chainId === sepolia.id;

  const metaMaskConnector = useMemo(
    () => connectors.find((c) => c.id === "metaMask") ?? connectors[0],
    [connectors],
  );

  const { data: balance } = useBalance({
    address,
    query: { enabled: Boolean(address) },
  });

  if (!isConnected) {
    return (
      <div>
        <button
          disabled={isConnecting}
          onClick={() => connect({ connector: metaMaskConnector })}
        >
          Connect MetaMask
        </button>
        {error && <div style={{ color: "crimson" }}>{error.message}</div>}
      </div>
    );
  }

  return (
    <div>
      <div>Address: {address}</div>
      <div>ChainId: {chainId}</div>
      {!isOnSepolia && (
        <button
          disabled={isSwitching}
          onClick={() => switchChain({ chainId: sepolia.id })}
        >
          Switch to Sepolia
        </button>
      )}
      <button onClick={() => disconnect()}>Disconnect</button>
      <div>
        Balance: {balance?.formatted} {balance?.symbol}
      </div>
    </div>
  );
}
```

---

## 9) Read contract data (contract balance, funders, contribution)

### 9.1. Contract balance (tổng ETH đã fund)

Dùng `useBalance` với **address = contract address**:

```ts
import { useBalance } from "wagmi";
import { contractAddr } from "./contracts/contractData";

const { data: contractBalance } = useBalance({ address: contractAddr });
```

### 9.2. Funders length

```ts
import { useReadContract } from "wagmi";
import { sepolia } from "wagmi/chains";
import { contractAddr } from "./contracts/contractData";
import { crowdfundingAbi } from "./contracts/crowdfundingAbi";

const { data: fundersLength } = useReadContract({
  address: contractAddr,
  abi: crowdfundingAbi,
  functionName: "getFundersLength",
  chainId: sepolia.id,
});
```

`fundersLength` sẽ là `bigint` → display: `String(fundersLength ?? 0n)`.

### 9.3. Contribution của user hiện tại

```ts
import { useAccount, useReadContract } from "wagmi";

const { address } = useAccount();

const { data: myContribution } = useReadContract({
  address: contractAddr,
  abi: crowdfundingAbi,
  functionName: "s_funderToAmount",
  args: address ? [address] : undefined,
  chainId: sepolia.id,
  query: { enabled: Boolean(address) },
});
```

---

## 10) Write contract (fund + withdraw) và chờ confirm

### 10.1. Fund (payable)

`fund()` không có args nhưng có `value`.

```ts
import { useWriteContract, useWaitForTransactionReceipt } from "wagmi";
import { parseEther } from "viem";

const { writeContract, data: hash, isPending } = useWriteContract();

const { isLoading: isConfirming, isSuccess } = useWaitForTransactionReceipt({
  hash,
  query: { enabled: Boolean(hash) },
});

const onFund = () => {
  writeContract({
    address: contractAddr,
    abi: crowdfundingAbi,
    functionName: "fund",
    value: parseEther("0.01"),
    chainId: sepolia.id,
  });
};
```

Giải thích:

- `useWriteContract()` gửi tx
- `useWaitForTransactionReceipt()` chờ tx “lên block”
- `value: parseEther('0.01')` là số ETH bạn fund

> Nếu tx revert `no available amount`, hãy tăng số ETH (vì cần >= 5 USD theo Chainlink feed).

### 10.2. Withdraw (onlyOwner)

```ts
const onWithdraw = () => {
  writeContract({
    address: contractAddr,
    abi: crowdfundingAbi,
    functionName: "withdraw",
    chainId: sepolia.id,
  });
};
```

Nếu bạn không phải `owner()`, tx sẽ revert với lỗi kiểu `OwnableUnauthorizedAccount`.

---

## 11) Events: watch realtime + fetch history

### 11.1. Watch realtime

```ts
import { useWatchContractEvent } from "wagmi";

useWatchContractEvent({
  address: contractAddr,
  abi: crowdfundingAbi,
  eventName: "Funded",
  chainId: sepolia.id,
  onLogs: (logs) => {
    // logs[i].args.funder / logs[i].args.value
    console.log("Funded logs", logs);
  },
});

useWatchContractEvent({
  address: contractAddr,
  abi: crowdfundingAbi,
  eventName: "Withdrawn",
  chainId: sepolia.id,
  onLogs: (logs) => {
    console.log("Withdrawn logs", logs);
  },
});
```

### 11.2. Fetch event history (query logs)

```ts
import { usePublicClient } from "wagmi";
import { parseAbiItem } from "viem";
import { deployBlock } from "./contracts/contractData";

const publicClient = usePublicClient({ chainId: sepolia.id });

async function fetchHistory() {
  if (!publicClient) return;

  const fundedEvent = parseAbiItem(
    "event Funded(address indexed funder, uint256 value)",
  );
  const withdrawnEvent = parseAbiItem("event Withdrawn(uint256 value)");

  const [fundedLogs, withdrawnLogs] = await Promise.all([
    publicClient.getLogs({
      address: contractAddr,
      event: fundedEvent,
      fromBlock: deployBlock,
      toBlock: "latest",
    }),
    publicClient.getLogs({
      address: contractAddr,
      event: withdrawnEvent,
      fromBlock: deployBlock,
      toBlock: "latest",
    }),
  ]);

  return { fundedLogs, withdrawnLogs };
}
```

Giải thích:

- `fromBlock = deployBlock` giúp query nhanh, tránh quét cả chain
- `parseAbiItem(...)` giúp viem decode args của event

---

## 12) Gợi ý map lại UI hiện tại của bạn

UI trong `web3-interface/src/App.tsx` đang có:

- Header có nút “Connect Wallet”
- 2 card: “Total Amount Funding” và “Funders”

Bạn có thể thay dữ liệu cứng bằng:

- `Total Amount Funding` = `useBalance({ address: contractAddr })`
- `Funders` = `useReadContract(getFundersLength)`

Và thêm 2 button:

- Fund 0.01 ETH
- Withdraw (chỉ owner)

---

## 13) Run frontend

Trong `web3-interface/`:

```bash
npm run dev
```

Checklist khi lỗi:

- MetaMask đã ở **Sepolia** chưa?
- `VITE_SEPOLIA_RPC_URL` có hoạt động không?
- Address contract có đúng không?
- Fund bị revert `no available amount` → tăng `value`

---

## 14) Bạn muốn mình viết tiếp phần nào?

Chọn 1:

1. Mình viết luôn một `App.tsx` hoàn chỉnh (đúng UI hiện tại) dùng wagmi để connect + fund + withdraw + event list
2. Hướng dẫn chạy local với `anvil` + mock price feed (không cần Sepolia)
3. Tách code theo “service layer” (hooks) để dễ maintain
