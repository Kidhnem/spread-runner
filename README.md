<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>Spread Runner — WETH Arbitrage Terminal</title>
<script src="https://cdnjs.cloudflare.com/ajax/libs/ethers/5.7.2/ethers.umd.min.js"></script>
<style>
  :root{
    --bg:#0A0B0D;
    --panel:#111317;
    --panel-2:#15171C;
    --line:#22252B;
    --text:#E6E7EA;
    --muted:#8A8F98;
    --amber:#E8A33D;
    --amber-dim:#9c7229;
    --green:#3ECF8E;
    --red:#E5484D;
    --mono: ui-monospace, "SF Mono", "IBM Plex Mono", Menlo, Consolas, monospace;
    --sans: -apple-system, BlinkMacSystemFont, "Segoe UI", Helvetica, Arial, sans-serif;
  }
  *{box-sizing:border-box;}
  body{
    margin:0; background:var(--bg); color:var(--text);
    font-family:var(--sans); -webkit-font-smoothing:antialiased;
  }
  .wrap{max-width:920px; margin:0 auto; padding:28px 18px 60px;}

  /* Ticker */
  .ticker{
    border:1px solid var(--line); background:var(--panel);
    border-radius:10px; overflow:hidden; margin-bottom:22px;
    position:relative;
  }
  .ticker-inner{
    display:flex; gap:36px; white-space:nowrap; padding:10px 18px;
    font-family:var(--mono); font-size:12.5px; color:var(--amber);
    animation:scroll 14s linear infinite;
  }
  .ticker.idle .ticker-inner{animation-play-state:paused;}
  @keyframes scroll{
    0%{transform:translateX(0);}
    100%{transform:translateX(-50%);}
  }
  .ticker-inner span{opacity:.85;}
  .dot{display:inline-block; width:6px; height:6px; border-radius:50%; background:var(--muted); margin-right:8px;}
  .dot.live{background:var(--green); box-shadow:0 0 8px var(--green);}

  header{margin-bottom:22px;}
  h1{
    font-size:22px; letter-spacing:-0.01em; margin:0 0 4px;
    font-weight:650;
  }
  .sub{color:var(--muted); font-size:13.5px; line-height:1.5;}

  .card{
    background:var(--panel); border:1px solid var(--line);
    border-radius:12px; padding:18px 18px 20px; margin-bottom:16px;
  }
  .card h2{
    font-size:11px; text-transform:uppercase; letter-spacing:.08em;
    color:var(--muted); margin:0 0 14px; font-weight:600;
  }

  .row{display:flex; gap:10px; flex-wrap:wrap; align-items:center;}
  .row + .row{margin-top:12px;}

  button{
    font-family:var(--sans); font-size:13.5px; font-weight:600;
    padding:10px 16px; border-radius:8px; border:1px solid var(--line);
    background:var(--panel-2); color:var(--text); cursor:pointer;
    transition:border-color .15s, background .15s;
  }
  button:hover:not(:disabled){border-color:var(--amber-dim);}
  button:disabled{opacity:.4; cursor:not-allowed;}
  button.primary{background:var(--amber); color:#1a1305; border-color:var(--amber);}
  button.primary:hover:not(:disabled){background:#f2ad47;}
  button.danger{background:var(--red); color:#2a0808; border-color:var(--red);}
  button.ghost{background:transparent;}

  select, input[type=number]{
    font-family:var(--mono); font-size:13.5px; padding:9px 10px;
    border-radius:8px; border:1px solid var(--line); background:var(--panel-2);
    color:var(--text); width:100%;
  }
  label{display:block; font-size:11.5px; color:var(--muted); margin-bottom:6px; text-transform:uppercase; letter-spacing:.05em;}
  .field{flex:1; min-width:130px;}

  .addr{font-family:var(--mono); font-size:12.5px; color:var(--amber);}

  /* Spread display */
  .spread-grid{display:grid; grid-template-columns:1fr 1fr; gap:12px; margin-bottom:14px;}
  .dex-box{
    border:1px solid var(--line); border-radius:10px; padding:14px;
    background:var(--panel-2);
  }
  .dex-box .name{font-size:12px; color:var(--muted); margin-bottom:8px; text-transform:uppercase; letter-spacing:.05em;}
  .dex-box .price{font-family:var(--mono); font-size:19px; font-weight:600;}
  .dex-box.cheap{border-color:var(--green);}
  .dex-box.cheap .name{color:var(--green);}
  .dex-box.rich{border-color:var(--red);}
  .dex-box.rich .name{color:var(--red);}

  .verdict{
    border-radius:10px; padding:16px; text-align:center;
    font-family:var(--mono);
  }
  .verdict.pos{background:rgba(62,207,142,.08); border:1px solid var(--green);}
  .verdict.neg{background:rgba(229,72,77,.06); border:1px solid var(--line);}
  .verdict .big{font-size:24px; font-weight:700;}
  .verdict.pos .big{color:var(--green);}
  .verdict.neg .big{color:var(--muted);}
  .verdict .small{font-size:12px; color:var(--muted); margin-top:6px;}

  .log{
    font-family:var(--mono); font-size:12px; max-height:220px; overflow-y:auto;
    background:var(--panel-2); border:1px solid var(--line); border-radius:8px;
    padding:10px 12px;
  }
  .log-line{padding:3px 0; border-bottom:1px dashed var(--line); color:var(--muted);}
  .log-line:last-child{border-bottom:none;}
  .log-line.ok{color:var(--green);}
  .log-line.err{color:var(--red);}
  .log-line .t{opacity:.5; margin-right:8px;}

  .warn{
    border:1px solid var(--amber-dim); background:rgba(232,163,61,.06);
    border-radius:10px; padding:14px 16px; font-size:12.5px; line-height:1.6; color:#D8B77A;
    margin-bottom:16px;
  }
  .warn b{color:var(--amber);}

  .status{font-size:12.5px; color:var(--muted); font-family:var(--mono);}
  .status.on{color:var(--green);}
  .kv{display:flex; justify-content:space-between; font-family:var(--mono); font-size:12.5px; padding:4px 0; color:var(--muted);}
  .kv b{color:var(--text); font-weight:600;}
</style>
</head>
<body>
<div class="wrap">

  <div class="ticker idle" id="ticker">
    <div class="ticker-inner" id="tickerInner">
      <span><span class="dot" id="tickerDot"></span>SPREAD RUNNER — connect a wallet to begin scanning</span>
      <span><span class="dot"></span>SPREAD RUNNER — connect a wallet to begin scanning</span>
    </div>
  </div>

  <header>
    <h1>Spread Runner</h1>
    <div class="sub">Ethereum mainnet · Uniswap V2 ↔ SushiSwap two-leg arbitrage terminal. Reads live router quotes, sizes gas against the spread, and lets you fire both legs through your own MetaMask wallet.</div>
  </header>

  <div class="warn">
    <b>Read before using real funds.</b> This tool executes two ordinary on-chain swaps, one on each DEX — it is <b>not</b> a flash loan or an atomic bundle, so you carry the tokens between legs and price can move against you mid-sequence. Public mempool transactions like these are routinely front-run or back-run by MEV searchers running purpose-built infrastructure with sub-second latency; a spread visible here can vanish before your second transaction confirms. Every trade also pays real gas twice. Start small, and treat displayed "net profit" as an estimate, not a guarantee.
  </div>

  <div class="card">
    <h2>Wallet</h2>
    <div class="row">
      <button class="primary" id="connectBtn">Connect MetaMask</button>
      <span class="status" id="walletStatus">Not connected</span>
    </div>
    <div class="row" id="walletDetails" style="display:none;">
      <div class="kv" style="flex:1;"><span>Account</span><b class="addr" id="acctAddr">—</b></div>
    </div>
    <div class="row" id="balRow" style="display:none;">
      <div class="kv" style="flex:1;"><span>ETH balance</span><b id="ethBal">—</b></div>
      <div class="kv" style="flex:1;"><span>WETH balance</span><b id="wethBal">—</b></div>
    </div>
    <div class="row" id="wrapRow" style="display:none;">
      <div class="field">
        <label>Wrap ETH → WETH</label>
        <input type="number" id="wrapAmt" placeholder="0.1" step="0.01" min="0">
      </div>
      <button id="wrapBtn" style="align-self:flex-end;">Wrap</button>
    </div>
  </div>

  <div class="card">
    <h2>Scan configuration</h2>
    <div class="row">
      <div class="field">
        <label>Pair (against WETH)</label>
        <select id="pairSelect">
          <option value="USDC">USDC</option>
          <option value="DAI">DAI</option>
        </select>
      </div>
      <div class="field">
        <label>Trade size (WETH)</label>
        <input type="number" id="sizeInput" value="0.5" step="0.01" min="0.001">
      </div>
      <div class="field">
        <label>Slippage tolerance</label>
        <select id="slippageSelect">
          <option value="0.3">0.3%</option>
          <option value="0.5" selected>0.5%</option>
          <option value="1">1%</option>
          <option value="2">2%</option>
        </select>
      </div>
    </div>
    <div class="row">
      <button id="scanOnceBtn" disabled>Scan once</button>
      <button id="autoScanBtn" disabled>Start auto-scan (10s)</button>
      <span class="status" id="scanStatus"></span>
    </div>
  </div>

  <div class="card" id="resultsCard" style="display:none;">
    <h2>Live quote</h2>
    <div class="spread-grid">
      <div class="dex-box" id="uniBox">
        <div class="name">Uniswap V2</div>
        <div class="price" id="uniPrice">—</div>
      </div>
      <div class="dex-box" id="sushiBox">
        <div class="name">SushiSwap</div>
        <div class="price" id="sushiPrice">—</div>
      </div>
    </div>
    <div class="verdict neg" id="verdict">
      <div class="big" id="verdictBig">—</div>
      <div class="small" id="verdictSmall">Run a scan to evaluate the round trip.</div>
    </div>
    <div class="row" style="margin-top:14px;">
      <button class="primary" id="executeBtn" disabled>Execute round trip</button>
    </div>
  </div>

  <div class="card">
    <h2>Activity log</h2>
    <div class="log" id="log">
      <div class="log-line"><span class="t">--:--:--</span>Waiting for wallet connection…</div>
    </div>
  </div>
</div>

<script>
const { ethers } = window;

/* ---------- Verified mainnet addresses ---------- */
const TOKENS = {
  WETH: { address: "0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2", decimals: 18, symbol: "WETH" },
  USDC: { address: "0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48", decimals: 6,  symbol: "USDC" },
  DAI:  { address: "0x6B175474E89094C44Da98b954EedeAC495271d0F", decimals: 18, symbol: "DAI"  }
};
const DEXES = {
  uniswap:   { name: "Uniswap V2", router: "0x7a250d5630B4cF539739dF2C5dAcb4c659F2488D" },
  sushiswap: { name: "SushiSwap",  router: "0xd9e1cE17f2641f24aE83637ab66a2cca9C378B9F" }
};
const MAINNET_CHAIN_ID = 1;

const ROUTER_ABI = [
  "function getAmountsOut(uint amountIn, address[] calldata path) external view returns (uint[] memory amounts)",
  "function swapExactTokensForTokens(uint amountIn, uint amountOutMin, address[] calldata path, address to, uint deadline) external returns (uint[] memory amounts)"
];
const ERC20_ABI = [
  "function balanceOf(address) view returns (uint256)",
  "function allowance(address owner, address spender) view returns (uint256)",
  "function approve(address spender, uint256 amount) returns (bool)"
];
const WETH_ABI = ERC20_ABI.concat([
  "function deposit() payable",
  "function withdraw(uint256)"
]);

/* ---------- State ---------- */
let provider, signer, account;
let autoScanTimer = null;
let lastQuote = null;

/* ---------- DOM ---------- */
const $ = id => document.getElementById(id);
const logEl = $("log");

function log(msg, cls) {
  const t = new Date().toLocaleTimeString();
  const line = document.createElement("div");
  line.className = "log-line" + (cls ? " " + cls : "");
  line.innerHTML = `<span class="t">${t}</span>${msg}`;
  logEl.prepend(line);
}

function fmt(n, d = 4) {
  return Number(n).toLocaleString(undefined, { maximumFractionDigits: d });
}

function setTicker(text, live) {
  const inner = $("tickerInner");
  inner.innerHTML = `<span><span class="dot ${live ? "live" : ""}"></span>${text}</span>` +
                     `<span><span class="dot ${live ? "live" : ""}"></span>${text}</span>`;
  $("ticker").classList.toggle("idle", !live);
}

/* ---------- Wallet connect ---------- */
$("connectBtn").addEventListener("click", connectWallet);

async function connectWallet() {
  if (!window.ethereum) {
    log("No injected wallet found. Install MetaMask and reload.", "err");
    alert("MetaMask (or another injected wallet) was not detected in this browser.");
    return;
  }
  try {
    provider = new ethers.providers.Web3Provider(window.ethereum, "any");
    await provider.send("eth_requestAccounts", []);
    signer = provider.getSigner();
    account = await signer.getAddress();

    const network = await provider.getNetwork();
    if (network.chainId !== MAINNET_CHAIN_ID) {
      log(`Connected wallet is on chain ${network.chainId}, not Ethereum mainnet (1). Switch networks in MetaMask.`, "err");
      $("walletStatus").textContent = `Wrong network (chainId ${network.chainId})`;
      $("walletStatus").classList.remove("on");
      return;
    }

    $("walletStatus").textContent = "Connected · mainnet";
    $("walletStatus").classList.add("on");
    $("acctAddr").textContent = account;
    $("walletDetails").style.display = "flex";
    $("balRow").style.display = "flex";
    $("wrapRow").style.display = "flex";
    $("scanOnceBtn").disabled = false;
    $("autoScanBtn").disabled = false;
    $("connectBtn").textContent = "Connected";
    $("connectBtn").disabled = true;

    window.ethereum.on("accountsChanged", () => window.location.reload());
    window.ethereum.on("chainChanged", () => window.location.reload());

    log(`Wallet connected: ${account}`, "ok");
    await refreshBalances();
  } catch (err) {
    log("Connection failed: " + (err.message || err), "err");
  }
}

async function refreshBalances() {
  try {
    const ethBal = await provider.getBalance(account);
    const weth = new ethers.Contract(TOKENS.WETH.address, ERC20_ABI, provider);
    const wethBal = await weth.balanceOf(account);
    $("ethBal").textContent = fmt(ethers.utils.formatEther(ethBal), 5) + " ETH";
    $("wethBal").textContent = fmt(ethers.utils.formatUnits(wethBal, 18), 5) + " WETH";
  } catch (err) {
    log("Balance fetch failed: " + (err.message || err), "err");
  }
}

/* ---------- Wrap ETH ---------- */
$("wrapBtn").addEventListener("click", async () => {
  const amt = $("wrapAmt").value;
  if (!amt || Number(amt) <= 0) return;
  try {
    $("wrapBtn").disabled = true;
    const weth = new ethers.Contract(TOKENS.WETH.address, WETH_ABI, signer);
    log(`Wrapping ${amt} ETH → WETH…`);
    const tx = await weth.deposit({ value: ethers.utils.parseEther(amt) });
    log(`Wrap tx sent: ${tx.hash}`);
    await tx.wait();
    log(`Wrap confirmed.`, "ok");
    await refreshBalances();
  } catch (err) {
    log("Wrap failed: " + (err.message || err), "err");
  } finally {
    $("wrapBtn").disabled = false;
  }
});

/* ---------- Scanning ---------- */
$("scanOnceBtn").addEventListener("click", () => scan());
$("autoScanBtn").addEventListener("click", toggleAutoScan);

function toggleAutoScan() {
  if (autoScanTimer) {
    clearInterval(autoScanTimer);
    autoScanTimer = null;
    $("autoScanBtn").textContent = "Start auto-scan (10s)";
    $("scanStatus").textContent = "Auto-scan stopped.";
    setTicker("Auto-scan paused", false);
  } else {
    scan();
    autoScanTimer = setInterval(scan, 10000);
    $("autoScanBtn").textContent = "Stop auto-scan";
    setTicker("Auto-scan running — polling Uniswap V2 & SushiSwap every 10s", true);
  }
}

async function getQuote(routerAddr, amountIn, path) {
  const router = new ethers.Contract(routerAddr, ROUTER_ABI, provider);
  const amounts = await router.getAmountsOut(amountIn, path);
  return amounts[amounts.length - 1];
}

async function scan() {
  if (!provider) return;
  $("resultsCard").style.display = "block";
  $("scanStatus").textContent = "Scanning…";

  try {
    const pairSym = $("pairSelect").value;
    const token = TOKENS[pairSym];
    const size = $("sizeInput").value || "0.1";
    const amountInWei = ethers.utils.parseUnits(size, TOKENS.WETH.decimals);

    const pathOut = [TOKENS.WETH.address, token.address];
    const pathBack = [token.address, TOKENS.WETH.address];

    // Quote WETH->token on both DEXs (this is the "buy leg" price)
    const [uniTokenOut, sushiTokenOut] = await Promise.all([
      getQuote(DEXES.uniswap.router, amountInWei, pathOut),
      getQuote(DEXES.sushiswap.router, amountInWei, pathOut)
    ]);

    const uniPrice = Number(ethers.utils.formatUnits(uniTokenOut, token.decimals)) / Number(size);
    const sushiPrice = Number(ethers.utils.formatUnits(sushiTokenOut, token.decimals)) / Number(size);

    $("uniPrice").textContent = `${fmt(uniPrice, 2)} ${token.symbol}`;
    $("sushiPrice").textContent = `${fmt(sushiPrice, 2)} ${token.symbol}`;
    $("uniBox").className = "dex-box" + (uniPrice > sushiPrice ? " cheap" : uniPrice < sushiPrice ? " rich" : "");
    $("sushiBox").className = "dex-box" + (sushiPrice > uniPrice ? " cheap" : sushiPrice < uniPrice ? " rich" : "");

    // Determine buy (higher token-out = better price) and sell dex, then quote the round trip
    const buyDex = uniTokenOut.gt(sushiTokenOut) ? "uniswap" : "sushiswap";
    const sellDex = buyDex === "uniswap" ? "sushiswap" : "uniswap";
    const tokenAcquired = buyDex === "uniswap" ? uniTokenOut : sushiTokenOut;

    const wethBack = await getQuote(DEXES[sellDex].router, tokenAcquired, pathBack);
    const grossProfitWei = wethBack.sub(amountInWei);
    const grossProfitEth = Number(ethers.utils.formatEther(grossProfitWei));

    // Gas estimate: 2 swaps + up to 2 approvals, mainnet gas price
    const feeData = await provider.getFeeData();
    const gasPrice = feeData.maxFeePerGas || feeData.gasPrice || ethers.BigNumber.from("30000000000");
    const estGasUnits = ethers.BigNumber.from("430000"); // 2 approvals + 2 swaps, conservative
    const estGasCostWei = gasPrice.mul(estGasUnits);
    const estGasCostEth = Number(ethers.utils.formatEther(estGasCostWei));

    const netProfitEth = grossProfitEth - estGasCostEth;

    lastQuote = {
      pairSym, token, size, amountInWei, buyDex, sellDex, tokenAcquired,
      grossProfitEth, estGasCostEth, netProfitEth,
      slippage: Number($("slippageSelect").value)
    };

    const verdictEl = $("verdict");
    if (netProfitEth > 0) {
      verdictEl.className = "verdict pos";
      $("verdictBig").textContent = `+${fmt(netProfitEth, 5)} WETH net`;
      $("verdictSmall").textContent =
        `Buy on ${DEXES[buyDex].name}, sell on ${DEXES[sellDex].name}. Gross ${fmt(grossProfitEth,5)} WETH − est. gas ${fmt(estGasCostEth,5)} WETH.`;
      $("executeBtn").disabled = false;
    } else {
      verdictEl.className = "verdict neg";
      $("verdictBig").textContent = `${fmt(netProfitEth, 5)} WETH net`;
      $("verdictSmall").textContent =
        `Spread doesn't clear estimated gas (${fmt(estGasCostEth,5)} WETH). No action recommended.`;
      $("executeBtn").disabled = true;
    }

    $("scanStatus").textContent = `Last scan: ${new Date().toLocaleTimeString()}`;
    setTicker(
      `${token.symbol}/WETH · Uni ${fmt(uniPrice,2)} · Sushi ${fmt(sushiPrice,2)} · net ${netProfitEth>0?"+":""}${fmt(netProfitEth,5)} WETH`,
      !!autoScanTimer
    );
  } catch (err) {
    $("scanStatus").textContent = "Scan failed.";
    log("Scan failed: " + (err.message || err), "err");
  }
}

/* ---------- Execute ---------- */
$("executeBtn").addEventListener("click", executeRoundTrip);

async function ensureAllowance(tokenAddr, spender, amount) {
  const token = new ethers.Contract(tokenAddr, ERC20_ABI, signer);
  const current = await token.allowance(account, spender);
  if (current.gte(amount)) return;
  log(`Approving ${spender} to spend token ${tokenAddr.slice(0,8)}…`);
  const tx = await token.approve(spender, ethers.constants.MaxUint256);
  log(`Approval tx sent: ${tx.hash}`);
  await tx.wait();
  log(`Approval confirmed.`, "ok");
}

async function executeRoundTrip() {
  if (!lastQuote) return;
  const { token, buyDex, sellDex, amountInWei, tokenAcquired, slippage } = lastQuote;
  const execBtn = $("executeBtn");
  execBtn.disabled = true;

  try {
    const deadline = Math.floor(Date.now() / 1000) + 600;
    const slipFactor = 1 - slippage / 100;

    // Confirm on-chain WETH balance covers the trade
    const weth = new ethers.Contract(TOKENS.WETH.address, ERC20_ABI, provider);
    const wethBalBefore = await weth.balanceOf(account);
    if (wethBalBefore.lt(amountInWei)) {
      log(`Insufficient WETH balance for this trade size. Wrap more ETH first.`, "err");
      execBtn.disabled = false;
      return;
    }

    // Leg 1: buy token on the cheaper DEX
    log(`Leg 1 — swapping ${lastQuote.size} WETH → ${token.symbol} on ${DEXES[buyDex].name}…`);
    await ensureAllowance(TOKENS.WETH.address, DEXES[buyDex].router, amountInWei);
    const minTokenOut = tokenAcquired.mul(Math.floor(slipFactor * 1000)).div(1000);
    const routerBuy = new ethers.Contract(DEXES[buyDex].router, ROUTER_ABI, signer);
    const tx1 = await routerBuy.swapExactTokensForTokens(
      amountInWei, minTokenOut, [TOKENS.WETH.address, token.address], account, deadline
    );
    log(`Leg 1 tx sent: ${tx1.hash}`);
    await tx1.wait();
    log(`Leg 1 confirmed.`, "ok");

    // Read actual token balance received
    const tokenContract = new ethers.Contract(token.address, ERC20_ABI, provider);
    const tokenBal = await tokenContract.balanceOf(account);

    // Leg 2: sell token on the pricier DEX
    log(`Leg 2 — swapping ${token.symbol} → WETH on ${DEXES[sellDex].name}…`);
    await ensureAllowance(token.address, DEXES[sellDex].router, tokenBal);
    const routerSell = new ethers.Contract(DEXES[sellDex].router, ROUTER_ABI, signer);
    const quotedBack = await getQuote(DEXES[sellDex].router, tokenBal, [token.address, TOKENS.WETH.address]);
    const minWethOut = quotedBack.mul(Math.floor(slipFactor * 1000)).div(1000);
    const tx2 = await routerSell.swapExactTokensForTokens(
      tokenBal, minWethOut, [token.address, TOKENS.WETH.address], account, deadline
    );
    log(`Leg 2 tx sent: ${tx2.hash}`);
    await tx2.wait();
    log(`Leg 2 confirmed.`, "ok");

    const wethBalAfter = await weth.balanceOf(account);
    const realized = Number(ethers.utils.formatEther(wethBalAfter.sub(wethBalBefore)));
    log(`Round trip complete. Realized change: ${realized > 0 ? "+" : ""}${fmt(realized, 6)} WETH (before gas already spent from ETH balance).`, realized > 0 ? "ok" : "err");
    await refreshBalances();
  } catch (err) {
    log("Execution failed or rejected: " + (err.message || err), "err");
  } finally {
    execBtn.disabled = false;
  }
}
</script>
</body>
</html>
