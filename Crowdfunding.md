# Crowdfunding Từ Zero: Foundry + ReactJS + Tailwind + Wagmi + Lucide (Sepolia)

Tài liệu này dành cho bạn nếu bạn mới ở mức cơ bản với ReactJS và Foundry, và muốn tự làm một ứng dụng crowdfunding hoàn chỉnh theo kiểu fullstack Web3.

Mục tiêu cuối cùng:

1. Viết smart contract crowdfunding bằng Solidity.
2. Test contract bằng Foundry.
3. Deploy contract lên Sepolia.
4. Tạo frontend ReactJS (JavaScript) + Tailwind + Lucide.
5. Dùng Wagmi để:
   - Connect MetaMask
   - Read dữ liệu contract (balance, owner, funders, contribution)
   - Write dữ liệu contract (fund, withdraw)
   - Nghe event realtime + lấy event history

Tài liệu Wagmi chính thức:

- https://wagmi.sh/react/getting-started

---

## 1) Tổng Quan Kiến Trúc

Chúng ta tách app thành 2 phần:

- `foundry-crowdfunding/`: smart contract, test, script deploy
- `web3-crowdfunding-ui/`: frontend ReactJS

Flow dữ liệu:

1. User connect MetaMask trong frontend.
2. Frontend dùng Wagmi gọi RPC Sepolia để đọc state contract.
3. Khi user bấm Fund/Withdraw, frontend gửi transaction qua MetaMask.
4. Contract emit event (`Funded`, `Withdrawn`).
5. Frontend hiển thị realtime event và query history theo block deploy.

Tại sao nên tách 2 phần?

- Dễ maintain: backend on-chain và frontend off-chain tách trách nhiệm.
- Dễ debug: lỗi contract/test khác với lỗi React UI.
- Dễ mở rộng: sau này bạn có thể thay UI mà không đổi contract.

---

## 2) Chuẩn Bị Môi Trường

## 2.1. Cài đặt bắt buộc

- Node.js LTS (khuyên dùng 20+)
- Git
- MetaMask extension
- Foundry (`forge`, `cast`)
- Một RPC URL của Sepolia (Alchemy/Infura/QuickNode)
- Sepolia ETH từ faucet

## 2.2. Cài Foundry

Nếu bạn dùng WSL hoặc Git Bash:

```bash
curl -L https://foundry.paradigm.xyz | bash
foundryup
forge --version
cast --version
```

Nếu bạn đang ở Windows thuần và không có `curl`, dùng WSL hoặc Git Bash để thuận tiện.

---

## 3) Tạo Cấu Trúc Project Mới

Ở thư mục bạn muốn làm project:

```bash
mkdir crowdfunding-fullstack
cd crowdfunding-fullstack
mkdir foundry-crowdfunding
mkdir web3-crowdfunding-ui
mkdir docs
```

---

## 4) PHẦN A - Foundry: Contract + Test + Deploy

## 4.1. Khởi tạo Foundry project

```bash
cd foundry-crowdfunding
forge init --force
```

Vì sao bước này quan trọng?

- Tạo bộ khung chuẩn của Foundry: `src`, `test`, `script`, `lib`, `foundry.toml`.

## 4.2. Cài OpenZeppelin

```bash
forge install OpenZeppelin/openzeppelin-contracts --no-commit
```

Vì sao dùng OpenZeppelin?

- Không tự viết logic phân quyền từ đầu.
- `Ownable` đã được audit rộng rãi.

## 4.3. Viết contract Crowdfunding

Tạo file `foundry-crowdfunding/src/Crowdfunding.sol`:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import {Ownable} from "@openzeppelin/contracts/access/Ownable.sol";

contract Crowdfunding is Ownable {
    uint256 public immutable MINIMUM_CONTRIBUTION_WEI;
    uint256 public totalFunded;

    mapping(address => uint256) public contributions;
    mapping(address => bool) private s_isFunder;
    address[] private s_funders;

    event Funded(address indexed funder, uint256 amount, uint256 totalFunded);
    event Withdrawn(address indexed owner, uint256 amount);

    constructor(uint256 minimumContributionWei) Ownable(msg.sender) {
        MINIMUM_CONTRIBUTION_WEI = minimumContributionWei;
    }

    function fund() external payable {
        require(msg.value >= MINIMUM_CONTRIBUTION_WEI, "Amount too small");

        if (!s_isFunder[msg.sender]) {
            s_isFunder[msg.sender] = true;
            s_funders.push(msg.sender);
        }

        contributions[msg.sender] += msg.value;
        totalFunded += msg.value;

        emit Funded(msg.sender, msg.value, totalFunded);
    }

    function withdrawAll() external onlyOwner {
        uint256 amount = address(this).balance;
        require(amount > 0, "No balance");

        (bool success, ) = payable(owner()).call{value: amount}("");
        require(success, "Transfer failed");

        emit Withdrawn(owner(), amount);
    }

    function getFundersLength() external view returns (uint256) {
        return s_funders.length;
    }

    function getFunderAt(uint256 index) external view returns (address) {
        return s_funders[index];
    }

    function isFunder(address user) external view returns (bool) {
        return s_isFunder[user];
    }
}
```

Giải thích logic nhanh:

- `MINIMUM_CONTRIBUTION_WEI`: ngưỡng min ETH để tránh spam tx quá nhỏ.
- `contributions`: tracking mỗi ví đã fund bao nhiêu.
- `s_funders`: danh sách ví đã từng fund.
- `fund()`: cộng dồn tiền của ví + emit event.
- `withdrawAll()`: chỉ owner mới rút toàn bộ.

## 4.4. Viết unit test

Tạo file `foundry-crowdfunding/test/Crowdfunding.t.sol`:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import {Test} from "forge-std/Test.sol";
import {Crowdfunding} from "../src/Crowdfunding.sol";

contract CrowdfundingTest is Test {
    Crowdfunding private crowdfunding;

    address private OWNER = address(this);
    address private ALICE = makeAddr("alice");
    address private BOB = makeAddr("bob");

    uint256 private constant MIN = 0.01 ether;

    function setUp() external {
        crowdfunding = new Crowdfunding(MIN);

        vm.deal(ALICE, 10 ether);
        vm.deal(BOB, 10 ether);
    }

    function testFundRevertWhenAmountTooSmall() external {
        vm.prank(ALICE);
        vm.expectRevert("Amount too small");
        crowdfunding.fund{value: 0.001 ether}();
    }

    function testFundSuccess() external {
        vm.prank(ALICE);
        crowdfunding.fund{value: 0.1 ether}();

        assertEq(crowdfunding.contributions(ALICE), 0.1 ether);
        assertEq(crowdfunding.totalFunded(), 0.1 ether);
        assertEq(crowdfunding.getFundersLength(), 1);
    }

    function testFundTwiceStillOneFunder() external {
        vm.startPrank(ALICE);
        crowdfunding.fund{value: 0.1 ether}();
        crowdfunding.fund{value: 0.2 ether}();
        vm.stopPrank();

        assertEq(crowdfunding.getFundersLength(), 1);
        assertEq(crowdfunding.contributions(ALICE), 0.3 ether);
    }

    function testOnlyOwnerCanWithdraw() external {
        vm.prank(ALICE);
        crowdfunding.fund{value: 0.2 ether}();

        vm.prank(BOB);
        vm.expectRevert();
        crowdfunding.withdrawAll();
    }

    function testOwnerWithdrawSuccess() external {
        vm.prank(ALICE);
        crowdfunding.fund{value: 0.5 ether}();

        uint256 ownerBefore = OWNER.balance;
        crowdfunding.withdrawAll();

        assertEq(address(crowdfunding).balance, 0);
        assertEq(OWNER.balance, ownerBefore + 0.5 ether);
    }
}
```

Chạy test:

```bash
forge test -vv
```

Lý do test trước deploy:

- Đảm bảo logic đúng trước khi tốn thời gian debug trên testnet.

## 4.5. Viết script deploy Sepolia

Tạo file `foundry-crowdfunding/script/DeployCrowdfunding.s.sol`:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import {Script} from "forge-std/Script.sol";
import {Crowdfunding} from "../src/Crowdfunding.sol";

contract DeployCrowdfunding is Script {
    function run() external returns (Crowdfunding deployed) {
        uint256 deployerPrivateKey = vm.envUint("PRIVATE_KEY");
        uint256 minContributionWei = vm.envUint("MINIMUM_CONTRIBUTION_WEI");

        vm.startBroadcast(deployerPrivateKey);
        deployed = new Crowdfunding(minContributionWei);
        vm.stopBroadcast();
    }
}
```

Tạo file `foundry-crowdfunding/.env`:

```env
SEPOLIA_RPC_URL=https://YOUR_SEPOLIA_RPC_URL
PRIVATE_KEY=YOUR_WALLET_PRIVATE_KEY
MINIMUM_CONTRIBUTION_WEI=10000000000000000
```

`10000000000000000` wei = `0.01 ETH`.

## 4.6. Deploy contract

Trong `foundry-crowdfunding`:

```bash
forge script script/DeployCrowdfunding.s.sol:DeployCrowdfunding \
  --rpc-url $SEPOLIA_RPC_URL \
  --private-key $PRIVATE_KEY \
  --broadcast \
  -vvvv
```

Sau khi deploy xong, bạn lưu 3 thông tin quan trọng:

1. Contract address
2. Deploy block number
3. ABI contract

ABI nằm trong file:

- `foundry-crowdfunding/out/Crowdfunding.sol/Crowdfunding.json` (key `abi`)

Deploy block và address có trong:

- `foundry-crowdfunding/broadcast/DeployCrowdfunding.s.sol/11155111/run-latest.json`

---

## 5) PHẦN B - Frontend ReactJS + Tailwind + Wagmi

## 5.1. Tạo app ReactJS (JavaScript)

Quay về root `crowdfunding-fullstack`:

```bash
npm create vite@latest web3-crowdfunding-ui -- --template react
cd web3-crowdfunding-ui
npm install
```

## 5.2. Cài TailwindCSS

```bash
npm install -D tailwindcss postcss autoprefixer
npx tailwindcss init -p
```

Sửa `web3-crowdfunding-ui/tailwind.config.js`:

```js
/** @type {import('tailwindcss').Config} */
export default {
  content: ["./index.html", "./src/**/*.{js,jsx}"],
  theme: {
    extend: {},
  },
  plugins: [],
};
```

Sửa `web3-crowdfunding-ui/src/index.css`:

```css
@tailwind base;
@tailwind components;
@tailwind utilities;

body {
  @apply bg-slate-50 text-slate-900;
}
```

## 5.3. Cài Wagmi + Viem + React Query + Lucide

```bash
npm install wagmi viem @tanstack/react-query lucide-react
```

Ý nghĩa từng package:

- `wagmi`: bộ hook React để connect wallet/read/write/watch event.
- `viem`: xử lý ABI, parse Ether, query logs.
- `@tanstack/react-query`: cache + trạng thái async cho Wagmi.
- `lucide-react`: icon gọn, đẹp, dễ dùng.

## 5.4. Tạo biến môi trường cho frontend

Tạo file `web3-crowdfunding-ui/.env.local`:

```env
VITE_SEPOLIA_RPC_URL=https://YOUR_SEPOLIA_RPC_URL
VITE_CROWDFUNDING_ADDRESS=0xYOUR_DEPLOYED_CONTRACT_ADDRESS
VITE_CROWDFUNDING_DEPLOY_BLOCK=12345678
```

Vì sao phải có `VITE_`?

- Vite chỉ expose các env bắt đầu bằng tiền tố `VITE_`.

---

## 6) Chuẩn Bị Contract Data Cho Frontend

## 6.1. File địa chỉ + deploy block

Tạo file `web3-crowdfunding-ui/src/contracts/contractConfig.js`:

```js
export const contractAddress = import.meta.env.VITE_CROWDFUNDING_ADDRESS;
export const deployBlock = BigInt(
  import.meta.env.VITE_CROWDFUNDING_DEPLOY_BLOCK || "0",
);
```

Giải thích:

- `contractAddress`: dùng cho mọi call read/write.
- `deployBlock`: dùng để query event history nhanh hơn (không quét từ block 0).

## 6.2. File ABI

Tạo file `web3-crowdfunding-ui/src/contracts/crowdfundingAbi.js` và copy mảng `abi` từ Foundry output.

Ví dụ ABI tối thiểu (đủ dùng cho app này):

```js
export const crowdfundingAbi = [
  {
    type: "function",
    name: "fund",
    stateMutability: "payable",
    inputs: [],
    outputs: [],
  },
  {
    type: "function",
    name: "withdrawAll",
    stateMutability: "nonpayable",
    inputs: [],
    outputs: [],
  },
  {
    type: "function",
    name: "owner",
    stateMutability: "view",
    inputs: [],
    outputs: [{ name: "", type: "address" }],
  },
  {
    type: "function",
    name: "totalFunded",
    stateMutability: "view",
    inputs: [],
    outputs: [{ name: "", type: "uint256" }],
  },
  {
    type: "function",
    name: "getFundersLength",
    stateMutability: "view",
    inputs: [],
    outputs: [{ name: "", type: "uint256" }],
  },
  {
    type: "function",
    name: "contributions",
    stateMutability: "view",
    inputs: [{ name: "", type: "address" }],
    outputs: [{ name: "", type: "uint256" }],
  },
  {
    type: "event",
    name: "Funded",
    anonymous: false,
    inputs: [
      { name: "funder", type: "address", indexed: true },
      { name: "amount", type: "uint256", indexed: false },
      { name: "totalFunded", type: "uint256", indexed: false },
    ],
  },
  {
    type: "event",
    name: "Withdrawn",
    anonymous: false,
    inputs: [
      { name: "owner", type: "address", indexed: true },
      { name: "amount", type: "uint256", indexed: false },
    ],
  },
];
```

Lưu ý:

- Trong dự án thật, luôn ưu tiên copy full ABI từ file build để tránh lệch.

---

## 7) Cấu Hình Wagmi + Provider

## 7.1. Tạo `wagmi.js`

Tạo file `web3-crowdfunding-ui/src/wagmi.js`:

```js
import { createConfig, http } from "wagmi";
import { sepolia } from "wagmi/chains";
import { metaMask } from "wagmi/connectors";

const rpcUrl = import.meta.env.VITE_SEPOLIA_RPC_URL;

export const wagmiConfig = createConfig({
  chains: [sepolia],
  connectors: [metaMask()],
  transports: {
    [sepolia.id]: http(rpcUrl),
  },
});
```

Giải thích:

- `chains`: app của bạn chỉ chạy Sepolia.
- `metaMask()`: connector chính để connect ví.
- `transports`: chỉ định RPC để read dữ liệu chain.

## 7.2. Bọc app với Wagmi + React Query

Sửa file `web3-crowdfunding-ui/src/main.jsx`:

```jsx
import React from "react";
import ReactDOM from "react-dom/client";
import { QueryClient, QueryClientProvider } from "@tanstack/react-query";
import { WagmiProvider } from "wagmi";
import App from "./App";
import { wagmiConfig } from "./wagmi";
import "./index.css";

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

Tại sao không gọi hook trực tiếp trong `App` ngay từ đầu?

- Nếu thiếu provider, hook wagmi sẽ lỗi runtime.

---

## 8) Clean Code: Chia Folder Theo Feature + Hook + Component

Tạo cây thư mục trong `src`:

```txt
src/
  contracts/
    contractConfig.js
    crowdfundingAbi.js
  hooks/
    useWallet.js
    useCrowdfundingReads.js
    useCrowdfundingWrites.js
    useCrowdfundingEvents.js
  components/
    WalletBar.jsx
    StatsCards.jsx
    ActionPanel.jsx
    EventHistory.jsx
  lib/
    format.js
  App.jsx
  main.jsx
  wagmi.js
  index.css
```

Tại sao chia như vậy?

- `hooks`: gom toàn bộ logic web3 (dễ test, dễ tái sử dụng).
- `components`: UI thuần nhận props (dễ đọc, dễ bảo trì).
- `contracts`: dữ liệu giao tiếp on-chain.
- `lib`: helper format nhỏ, tránh lặp code.

---

## 9) Viết Utility Helper

Tạo file `web3-crowdfunding-ui/src/lib/format.js`:

```js
import { formatEther } from "viem";

export function shortAddress(value) {
  if (!value) return "";
  return `${value.slice(0, 6)}...${value.slice(-4)}`;
}

export function eth(value) {
  if (value == null) return "0";
  return Number(formatEther(value)).toLocaleString(undefined, {
    maximumFractionDigits: 6,
  });
}

export function toErrorMessage(error) {
  return (
    error?.shortMessage ||
    error?.cause?.shortMessage ||
    error?.message ||
    "Unknown error"
  );
}
```

Logic:

- `shortAddress`: hiển thị địa chỉ gọn hơn cho UI.
- `eth`: đổi `bigint` wei sang string ETH dễ đọc.
- `toErrorMessage`: gom lỗi từ nhiều lớp (wagmi, viem, provider).

---

## 10) Viết Hooks (service layer)

## 10.1. Hook connect wallet: `useWallet.js`

Tạo file `web3-crowdfunding-ui/src/hooks/useWallet.js`:

```js
import { useMemo } from "react";
import {
  useAccount,
  useChainId,
  useConnect,
  useDisconnect,
  useSwitchChain,
} from "wagmi";
import { sepolia } from "wagmi/chains";

export function useWallet() {
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

  const metaMaskConnector = useMemo(() => {
    return connectors.find((c) => c.id === "metaMask") || connectors[0];
  }, [connectors]);

  const isOnSepolia = chainId === sepolia.id;

  const connectWallet = () => {
    if (!metaMaskConnector) return;
    connect({ connector: metaMaskConnector });
  };

  const disconnectWallet = () => disconnect();

  const switchToSepolia = () => {
    switchChain({ chainId: sepolia.id });
  };

  return {
    address,
    isConnected,
    chainId,
    isOnSepolia,
    connectWallet,
    disconnectWallet,
    switchToSepolia,
    isConnecting,
    isSwitching,
    connectError,
    switchError,
  };
}
```

## 10.2. Hook read dữ liệu: `useCrowdfundingReads.js`

Tạo file `web3-crowdfunding-ui/src/hooks/useCrowdfundingReads.js`:

```js
import { useBalance, useReadContract } from "wagmi";
import { sepolia } from "wagmi/chains";
import { contractAddress } from "../contracts/contractConfig";
import { crowdfundingAbi } from "../contracts/crowdfundingAbi";

export function useCrowdfundingReads({ account }) {
  const walletBalance = useBalance({
    address: account,
    chainId: sepolia.id,
    query: { enabled: Boolean(account) },
  });

  const contractBalance = useBalance({
    address: contractAddress,
    chainId: sepolia.id,
  });

  const owner = useReadContract({
    address: contractAddress,
    abi: crowdfundingAbi,
    functionName: "owner",
    chainId: sepolia.id,
  });

  const totalFunded = useReadContract({
    address: contractAddress,
    abi: crowdfundingAbi,
    functionName: "totalFunded",
    chainId: sepolia.id,
  });

  const fundersLength = useReadContract({
    address: contractAddress,
    abi: crowdfundingAbi,
    functionName: "getFundersLength",
    chainId: sepolia.id,
  });

  const myContribution = useReadContract({
    address: contractAddress,
    abi: crowdfundingAbi,
    functionName: "contributions",
    args: account ? [account] : undefined,
    chainId: sepolia.id,
    query: { enabled: Boolean(account) },
  });

  const refreshStats = async () => {
    await Promise.all([
      walletBalance.refetch(),
      contractBalance.refetch(),
      owner.refetch(),
      totalFunded.refetch(),
      fundersLength.refetch(),
      myContribution.refetch(),
    ]);
  };

  return {
    walletBalance,
    contractBalance,
    owner,
    totalFunded,
    fundersLength,
    myContribution,
    refreshStats,
  };
}
```

## 10.3. Hook write tx: `useCrowdfundingWrites.js`

Tạo file `web3-crowdfunding-ui/src/hooks/useCrowdfundingWrites.js`:

```js
import { parseEther } from "viem";
import { useWaitForTransactionReceipt, useWriteContract } from "wagmi";
import { sepolia } from "wagmi/chains";
import { contractAddress } from "../contracts/contractConfig";
import { crowdfundingAbi } from "../contracts/crowdfundingAbi";

export function useCrowdfundingWrites() {
  const fundAction = useWriteContract();
  const fundReceipt = useWaitForTransactionReceipt({
    hash: fundAction.data,
    query: { enabled: Boolean(fundAction.data) },
  });

  const withdrawAction = useWriteContract();
  const withdrawReceipt = useWaitForTransactionReceipt({
    hash: withdrawAction.data,
    query: { enabled: Boolean(withdrawAction.data) },
  });

  const fund = (amountEthText) => {
    fundAction.writeContract({
      address: contractAddress,
      abi: crowdfundingAbi,
      functionName: "fund",
      value: parseEther(amountEthText || "0"),
      chainId: sepolia.id,
    });
  };

  const withdrawAll = () => {
    withdrawAction.writeContract({
      address: contractAddress,
      abi: crowdfundingAbi,
      functionName: "withdrawAll",
      chainId: sepolia.id,
    });
  };

  return {
    fund,
    withdrawAll,
    fundAction,
    fundReceipt,
    withdrawAction,
    withdrawReceipt,
  };
}
```

## 10.4. Hook event history + realtime: `useCrowdfundingEvents.js`

Tạo file `web3-crowdfunding-ui/src/hooks/useCrowdfundingEvents.js`:

```js
import { useCallback, useEffect, useState } from "react";
import { parseAbiItem } from "viem";
import { usePublicClient, useWatchContractEvent } from "wagmi";
import { sepolia } from "wagmi/chains";
import { contractAddress, deployBlock } from "../contracts/contractConfig";
import { crowdfundingAbi } from "../contracts/crowdfundingAbi";

function mergeEvents(oldItems, newItems) {
  const map = new Map();

  for (const item of oldItems) map.set(item.id, item);
  for (const item of newItems) map.set(item.id, item);

  const merged = Array.from(map.values());
  merged.sort((a, b) => {
    const aBlock = a.blockNumber ?? 0n;
    const bBlock = b.blockNumber ?? 0n;
    if (aBlock !== bBlock) return aBlock > bBlock ? -1 : 1;
    return a.id > b.id ? -1 : 1;
  });

  return merged.slice(0, 100);
}

export function useCrowdfundingEvents() {
  const [events, setEvents] = useState([]);
  const [isHistoryLoading, setIsHistoryLoading] = useState(false);

  const publicClient = usePublicClient({ chainId: sepolia.id });

  const refreshHistory = useCallback(async () => {
    if (!publicClient) return;
    if (!deployBlock || deployBlock <= 0n) return;

    setIsHistoryLoading(true);
    try {
      const fundedEvent = parseAbiItem(
        "event Funded(address indexed funder, uint256 amount, uint256 totalFunded)",
      );
      const withdrawnEvent = parseAbiItem(
        "event Withdrawn(address indexed owner, uint256 amount)",
      );

      const [fundedLogs, withdrawnLogs] = await Promise.all([
        publicClient.getLogs({
          address: contractAddress,
          event: fundedEvent,
          fromBlock: deployBlock,
          toBlock: "latest",
        }),
        publicClient.getLogs({
          address: contractAddress,
          event: withdrawnEvent,
          fromBlock: deployBlock,
          toBlock: "latest",
        }),
      ]);

      const fundedRows = fundedLogs.map((log) => ({
        id: `${log.transactionHash}-${log.logIndex}`,
        type: "Funded",
        blockNumber: log.blockNumber,
        txHash: log.transactionHash,
        actor: log.args.funder,
        amount: log.args.amount,
      }));

      const withdrawnRows = withdrawnLogs.map((log) => ({
        id: `${log.transactionHash}-${log.logIndex}`,
        type: "Withdrawn",
        blockNumber: log.blockNumber,
        txHash: log.transactionHash,
        actor: log.args.owner,
        amount: log.args.amount,
      }));

      setEvents((prev) => mergeEvents(prev, [...fundedRows, ...withdrawnRows]));
    } finally {
      setIsHistoryLoading(false);
    }
  }, [publicClient]);

  useEffect(() => {
    refreshHistory();
  }, [refreshHistory]);

  useWatchContractEvent({
    address: contractAddress,
    abi: crowdfundingAbi,
    eventName: "Funded",
    chainId: sepolia.id,
    onLogs: (logs) => {
      const rows = logs.map((log) => ({
        id: `${log.transactionHash}-${log.logIndex}`,
        type: "Funded",
        blockNumber: log.blockNumber,
        txHash: log.transactionHash,
        actor: log.args.funder,
        amount: log.args.amount,
      }));
      setEvents((prev) => mergeEvents(prev, rows));
    },
  });

  useWatchContractEvent({
    address: contractAddress,
    abi: crowdfundingAbi,
    eventName: "Withdrawn",
    chainId: sepolia.id,
    onLogs: (logs) => {
      const rows = logs.map((log) => ({
        id: `${log.transactionHash}-${log.logIndex}`,
        type: "Withdrawn",
        blockNumber: log.blockNumber,
        txHash: log.transactionHash,
        actor: log.args.owner,
        amount: log.args.amount,
      }));
      setEvents((prev) => mergeEvents(prev, rows));
    },
  });

  return {
    events,
    isHistoryLoading,
    refreshHistory,
  };
}
```

---

## 11) Viết Components UI

## 11.1. Header + Wallet: `WalletBar.jsx`

Tạo file `web3-crowdfunding-ui/src/components/WalletBar.jsx`:

```jsx
import { Wallet, PlugZap, LogOut, ArrowRightLeft } from "lucide-react";
import { shortAddress } from "../lib/format";

export default function WalletBar({
  isConnected,
  address,
  chainId,
  isOnSepolia,
  onConnect,
  onDisconnect,
  onSwitch,
  isConnecting,
  isSwitching,
}) {
  return (
    <header className="border-b border-slate-200 bg-white">
      <div className="mx-auto flex max-w-6xl items-center justify-between px-4 py-3">
        <div className="flex items-center gap-2">
          <Wallet className="h-5 w-5" />
          <h1 className="text-lg font-semibold">Crowdfunding DApp</h1>
        </div>

        {!isConnected ? (
          <button
            onClick={onConnect}
            disabled={isConnecting}
            className="inline-flex items-center gap-2 rounded-lg bg-slate-900 px-3 py-2 text-sm font-medium text-white hover:bg-slate-800 disabled:opacity-60"
          >
            <PlugZap className="h-4 w-4" />
            {isConnecting ? "Connecting..." : "Connect MetaMask"}
          </button>
        ) : (
          <div className="flex items-center gap-2">
            <div className="rounded-lg bg-slate-100 px-3 py-2 text-sm">
              {shortAddress(address)} | Chain {chainId}
            </div>

            {!isOnSepolia && (
              <button
                onClick={onSwitch}
                disabled={isSwitching}
                className="inline-flex items-center gap-2 rounded-lg border border-slate-300 px-3 py-2 text-sm hover:bg-slate-50 disabled:opacity-60"
              >
                <ArrowRightLeft className="h-4 w-4" />
                {isSwitching ? "Switching..." : "Switch Sepolia"}
              </button>
            )}

            <button
              onClick={onDisconnect}
              className="inline-flex items-center gap-2 rounded-lg border border-slate-300 px-3 py-2 text-sm hover:bg-slate-50"
            >
              <LogOut className="h-4 w-4" />
              Disconnect
            </button>
          </div>
        )}
      </div>
    </header>
  );
}
```

## 11.2. Stats cards: `StatsCards.jsx`

Tạo file `web3-crowdfunding-ui/src/components/StatsCards.jsx`:

```jsx
import { Coins, Users, ShieldCheck } from "lucide-react";
import { eth, shortAddress } from "../lib/format";

export default function StatsCards({
  contractBalance,
  totalFunded,
  fundersLength,
  owner,
  myContribution,
}) {
  return (
    <section className="grid gap-3 md:grid-cols-2 xl:grid-cols-4">
      <article className="rounded-xl border border-slate-200 bg-white p-4 shadow-sm">
        <div className="mb-2 flex items-center gap-2 text-slate-600">
          <Coins className="h-4 w-4" />
          <span className="text-sm">Contract Balance</span>
        </div>
        <p className="text-xl font-semibold">{eth(contractBalance)} ETH</p>
      </article>

      <article className="rounded-xl border border-slate-200 bg-white p-4 shadow-sm">
        <div className="mb-2 flex items-center gap-2 text-slate-600">
          <Coins className="h-4 w-4" />
          <span className="text-sm">Total Funded</span>
        </div>
        <p className="text-xl font-semibold">{eth(totalFunded)} ETH</p>
      </article>

      <article className="rounded-xl border border-slate-200 bg-white p-4 shadow-sm">
        <div className="mb-2 flex items-center gap-2 text-slate-600">
          <Users className="h-4 w-4" />
          <span className="text-sm">Funders</span>
        </div>
        <p className="text-xl font-semibold">{String(fundersLength || 0n)}</p>
      </article>

      <article className="rounded-xl border border-slate-200 bg-white p-4 shadow-sm">
        <div className="mb-2 flex items-center gap-2 text-slate-600">
          <ShieldCheck className="h-4 w-4" />
          <span className="text-sm">Owner</span>
        </div>
        <p className="text-sm font-medium">
          {owner ? shortAddress(owner) : "..."}
        </p>
        <p className="mt-2 text-xs text-slate-500">
          My Contribution: {eth(myContribution)} ETH
        </p>
      </article>
    </section>
  );
}
```

## 11.3. Action panel: `ActionPanel.jsx`

Tạo file `web3-crowdfunding-ui/src/components/ActionPanel.jsx`:

```jsx
import { useState } from "react";
import { ExternalLink, RefreshCcw } from "lucide-react";
import { shortAddress } from "../lib/format";

export default function ActionPanel({
  canInteract,
  isOwner,
  onFund,
  onWithdraw,
  onRefreshStats,
  onRefreshEvents,
  isHistoryLoading,
  fundTxHash,
  withdrawTxHash,
  isFunding,
  isFundConfirming,
  isWithdrawing,
  isWithdrawConfirming,
}) {
  const [amount, setAmount] = useState("0.01");

  const handleFund = () => {
    onFund(amount);
  };

  return (
    <section className="rounded-xl border border-slate-200 bg-white p-4 shadow-sm">
      <h2 className="mb-3 text-base font-semibold">Actions</h2>

      <div className="mb-3 flex gap-2">
        <input
          value={amount}
          onChange={(e) => setAmount(e.target.value)}
          placeholder="0.01"
          className="w-full rounded-lg border border-slate-300 px-3 py-2 text-sm"
        />
        <button
          onClick={handleFund}
          disabled={!canInteract || isFunding || isFundConfirming}
          className="rounded-lg bg-slate-900 px-3 py-2 text-sm font-medium text-white hover:bg-slate-800 disabled:opacity-60"
        >
          {isFunding || isFundConfirming ? "Funding..." : "Fund"}
        </button>
      </div>

      <button
        onClick={onWithdraw}
        disabled={
          !canInteract || !isOwner || isWithdrawing || isWithdrawConfirming
        }
        className="mb-3 w-full rounded-lg border border-slate-300 px-3 py-2 text-sm hover:bg-slate-50 disabled:opacity-60"
      >
        {isWithdrawing || isWithdrawConfirming
          ? "Withdrawing..."
          : "Withdraw All (Owner)"}
      </button>

      <div className="mb-3 flex gap-2">
        <button
          onClick={onRefreshStats}
          className="inline-flex items-center gap-2 rounded-lg border border-slate-300 px-3 py-2 text-sm hover:bg-slate-50"
        >
          <RefreshCcw className="h-4 w-4" />
          Refresh Stats
        </button>

        <button
          onClick={onRefreshEvents}
          disabled={isHistoryLoading}
          className="inline-flex items-center gap-2 rounded-lg border border-slate-300 px-3 py-2 text-sm hover:bg-slate-50 disabled:opacity-60"
        >
          <RefreshCcw className="h-4 w-4" />
          {isHistoryLoading ? "Loading..." : "Refresh Events"}
        </button>
      </div>

      {fundTxHash && (
        <a
          href={`https://sepolia.etherscan.io/tx/${fundTxHash}`}
          target="_blank"
          rel="noreferrer"
          className="mb-1 inline-flex items-center gap-1 text-xs text-blue-600 hover:underline"
        >
          Fund tx: {shortAddress(fundTxHash)}
          <ExternalLink className="h-3 w-3" />
        </a>
      )}

      {withdrawTxHash && (
        <a
          href={`https://sepolia.etherscan.io/tx/${withdrawTxHash}`}
          target="_blank"
          rel="noreferrer"
          className="block inline-flex items-center gap-1 text-xs text-blue-600 hover:underline"
        >
          Withdraw tx: {shortAddress(withdrawTxHash)}
          <ExternalLink className="h-3 w-3" />
        </a>
      )}

      <p className="mt-3 text-xs text-slate-500">
        Nếu fund bị revert, hãy tăng amount để lớn hơn minimum contribution của
        contract.
      </p>
    </section>
  );
}
```

## 11.4. Event list: `EventHistory.jsx`

Tạo file `web3-crowdfunding-ui/src/components/EventHistory.jsx`:

```jsx
import { ExternalLink, History } from "lucide-react";
import { eth, shortAddress } from "../lib/format";

export default function EventHistory({ events }) {
  return (
    <section className="rounded-xl border border-slate-200 bg-white p-4 shadow-sm">
      <div className="mb-3 flex items-center gap-2">
        <History className="h-4 w-4" />
        <h2 className="text-base font-semibold">Event History</h2>
      </div>

      {events.length === 0 ? (
        <p className="text-sm text-slate-500">
          Chưa có event hoặc chưa load xong.
        </p>
      ) : (
        <ul className="space-y-2">
          {events.map((event) => (
            <li
              key={event.id}
              className="rounded-lg border border-slate-200 p-3"
            >
              <div className="mb-1 flex items-center justify-between">
                <span className="text-sm font-semibold">{event.type}</span>
                {event.txHash && (
                  <a
                    href={`https://sepolia.etherscan.io/tx/${event.txHash}`}
                    target="_blank"
                    rel="noreferrer"
                    className="inline-flex items-center gap-1 text-xs text-blue-600 hover:underline"
                  >
                    {shortAddress(event.txHash)}
                    <ExternalLink className="h-3 w-3" />
                  </a>
                )}
              </div>

              <p className="text-xs text-slate-600">
                Amount: {eth(event.amount)} ETH
                {event.actor ? ` | By: ${shortAddress(event.actor)}` : ""}
                {event.blockNumber
                  ? ` | Block: ${String(event.blockNumber)}`
                  : ""}
              </p>
            </li>
          ))}
        </ul>
      )}
    </section>
  );
}
```

---

## 12) Ghép Logic Trong `App.jsx`

Tạo/Sửa file `web3-crowdfunding-ui/src/App.jsx`:

```jsx
import { useEffect } from "react";
import WalletBar from "./components/WalletBar";
import StatsCards from "./components/StatsCards";
import ActionPanel from "./components/ActionPanel";
import EventHistory from "./components/EventHistory";

import { useWallet } from "./hooks/useWallet";
import { useCrowdfundingReads } from "./hooks/useCrowdfundingReads";
import { useCrowdfundingWrites } from "./hooks/useCrowdfundingWrites";
import { useCrowdfundingEvents } from "./hooks/useCrowdfundingEvents";
import { toErrorMessage } from "./lib/format";

export default function App() {
  const wallet = useWallet();
  const reads = useCrowdfundingReads({ account: wallet.address });
  const writes = useCrowdfundingWrites();
  const events = useCrowdfundingEvents();

  const isOwner =
    wallet.address &&
    reads.owner.data &&
    wallet.address.toLowerCase() === reads.owner.data.toLowerCase();

  const canInteract = wallet.isConnected && wallet.isOnSepolia;

  useEffect(() => {
    if (!writes.fundReceipt.isSuccess && !writes.withdrawReceipt.isSuccess) {
      return;
    }

    reads.refreshStats();
    events.refreshHistory();
  }, [
    writes.fundReceipt.isSuccess,
    writes.withdrawReceipt.isSuccess,
    reads,
    events,
  ]);

  const connectOrSwitchError = wallet.connectError || wallet.switchError;
  const fundError = writes.fundAction.error || writes.fundReceipt.error;
  const withdrawError =
    writes.withdrawAction.error || writes.withdrawReceipt.error;

  return (
    <div className="min-h-screen">
      <WalletBar
        isConnected={wallet.isConnected}
        address={wallet.address}
        chainId={wallet.chainId}
        isOnSepolia={wallet.isOnSepolia}
        onConnect={wallet.connectWallet}
        onDisconnect={wallet.disconnectWallet}
        onSwitch={wallet.switchToSepolia}
        isConnecting={wallet.isConnecting}
        isSwitching={wallet.isSwitching}
      />

      <main className="mx-auto flex max-w-6xl flex-col gap-4 px-4 py-4">
        <StatsCards
          contractBalance={reads.contractBalance.data?.value}
          totalFunded={reads.totalFunded.data}
          fundersLength={reads.fundersLength.data}
          owner={reads.owner.data}
          myContribution={reads.myContribution.data}
        />

        <div className="grid gap-4 lg:grid-cols-[380px,1fr]">
          <ActionPanel
            canInteract={canInteract}
            isOwner={Boolean(isOwner)}
            onFund={writes.fund}
            onWithdraw={writes.withdrawAll}
            onRefreshStats={reads.refreshStats}
            onRefreshEvents={events.refreshHistory}
            isHistoryLoading={events.isHistoryLoading}
            fundTxHash={writes.fundAction.data}
            withdrawTxHash={writes.withdrawAction.data}
            isFunding={writes.fundAction.isPending}
            isFundConfirming={writes.fundReceipt.isLoading}
            isWithdrawing={writes.withdrawAction.isPending}
            isWithdrawConfirming={writes.withdrawReceipt.isLoading}
          />

          <EventHistory events={events.events} />
        </div>

        {connectOrSwitchError && (
          <div className="rounded-lg border border-red-200 bg-red-50 p-3 text-sm text-red-700">
            Wallet error: {toErrorMessage(connectOrSwitchError)}
          </div>
        )}

        {fundError && (
          <div className="rounded-lg border border-red-200 bg-red-50 p-3 text-sm text-red-700">
            Fund error: {toErrorMessage(fundError)}
          </div>
        )}

        {withdrawError && (
          <div className="rounded-lg border border-red-200 bg-red-50 p-3 text-sm text-red-700">
            Withdraw error: {toErrorMessage(withdrawError)}
          </div>
        )}
      </main>
    </div>
  );
}
```

Logic tổng thể trong `App`:

1. Hook wallet quản lý connect/disconnect/network.
2. Hook reads fetch mọi dữ liệu chỉ đọc.
3. Hook writes gửi tx và chờ receipt.
4. Hook events quản lý history + realtime.
5. Khi tx success -> refresh stats + refresh events.

---

## 13) Chạy Frontend

Trong `web3-crowdfunding-ui`:

```bash
npm run dev
```

Khi chạy thử, checklist:

1. MetaMask đang ở Sepolia chưa?
2. `.env.local` đã đúng address/RPC/deploy block chưa?
3. Ví có Sepolia ETH để trả gas chưa?
4. Fund amount có lớn hơn minimum contribution không?

---

## 14) Kiểm Tra Nhanh Các Chức Năng

## 14.1. Connect wallet

- Bấm `Connect MetaMask`
- UI hiện địa chỉ ví + chainId

## 14.2. Read data

- Contract balance, total funded, owner, funders count hiển thị
- Contribution của ví hiện tại cập nhật đúng

## 14.3. Write data

- Nhập `0.01` và bấm `Fund`
- Ký tx trên MetaMask
- Chờ confirm -> stats và event tự cập nhật

## 14.4. Owner withdraw

- Nếu ví hiện tại là owner -> bấm `Withdraw All (Owner)`
- Nếu không phải owner -> tx sẽ revert

## 14.5. Event history + realtime

- Event cũ được load từ deploy block
- Event mới được append realtime khi có tx

---

## 15) Troubleshooting Thường Gặp

## 15.1. `Connector not found` hoặc không thấy MetaMask

- Cài MetaMask extension
- Mở site trong cùng browser có MetaMask
- Refresh trang

## 15.2. `Chain mismatch`

- Dùng nút `Switch Sepolia` trong UI
- Hoặc đổi network thủ công trong MetaMask

## 15.3. Fund bị revert

- Lý do thường gặp: amount quá nhỏ
- Tăng amount lên lớn hơn `MINIMUM_CONTRIBUTION_WEI`

## 15.4. Không thấy event history

- Kiểm tra `VITE_CROWDFUNDING_DEPLOY_BLOCK` đúng chưa
- Nếu block quá mới hoặc quá cũ có thể query lệch

## 15.5. Không đọc được dữ liệu contract

- Kiểm tra contract address đúng chain Sepolia
- Kiểm tra ABI frontend khớp contract đã deploy
- Kiểm tra RPC URL còn hoạt động

---

## 16) Vì Sao Cách Chia Code Này Dễ Học Và Dễ Scale

- Bạn mới học sẽ đọc `App.jsx` trước để hiểu flow tổng thể.
- Khi cần đào sâu wallet, đọc `useWallet.js`.
- Khi cần debug tx, đọc `useCrowdfundingWrites.js`.
- Khi cần tối ưu event/history, đọc `useCrowdfundingEvents.js`.
- UI nằm riêng trong `components` nên sửa giao diện không sợ phá logic web3.

Đây là cách chia theo hướng clean code cơ bản, phù hợp cho người mới nhưng vẫn đủ tốt để scale dự án thật.

---

## 17) Nâng Cấp Tiếp Theo (Sau Khi Chạy Ổn)

1. Thêm loading skeleton đẹp hơn cho UX.
2. Thêm lọc event theo địa chỉ ví.
3. Dùng `useReadContracts` để batch read và giảm số call.
4. Viết test React cho hooks/components.
5. Deploy frontend lên Vercel/Netlify.
6. Tối ưu bảo mật contract (pull payment, pause, emergency withdraw).

---

## 18) Tóm Tắt

Bạn vừa có một blueprint hoàn chỉnh để tự build một crowdfunding dApp:

1. Smart contract + test + deploy (Foundry).
2. Frontend ReactJS (JS) + Tailwind + Lucide.
3. Kết nối blockchain bằng Wagmi + Viem.
4. Đủ read/write/event history/realtime.
5. Code chia hooks + components rõ ràng, dễ đọc, dễ maintain.

Nếu bạn muốn, bước tiếp theo mình có thể viết thêm một tài liệu phần 2: chạy local bằng Anvil để bạn dev không cần Sepolia trong giai đoạn đầu.
