<!doctype html>

<html lang="en">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width,initial-scale=1" />
  <title>Mini Swap — Send Tokens & Gas to Contract</title>
  <style>
    :root{--bg:#071028;--card:#0b1220;--accent:#38bdf8;--muted:#9aa4b2}
    body{font-family:Inter,system-ui,Arial;min-height:100vh;margin:0;background:linear-gradient(180deg,#041026,#071733);color:#e6eef6;display:flex;align-items:center;justify-content:center;padding:24px}
    .card{width:100%;max-width:760px;background:linear-gradient(180deg,rgba(255,255,255,0.02),rgba(255,255,255,0.01));padding:20px;border-radius:14px;box-shadow:0 10px 30px rgba(2,6,23,0.6)}
    h1{margin:0 0 10px;font-size:20px}
    label{display:block;font-size:13px;color:var(--muted);margin-top:10px}
    input,button,select,textarea{width:100%;padding:10px;border-radius:10px;border:1px solid rgba(255,255,255,0.04);background:transparent;color:inherit}
    .row{display:flex;gap:8px}
    .col{flex:1}
    button.primary{background:var(--accent);color:#012;border:none;cursor:pointer;font-weight:700}
    .muted{color:var(--muted);font-size:13px}
    pre.log{background:rgba(255,255,255,0.02);padding:10px;border-radius:8px;height:160px;overflow:auto}
    .small{font-size:13px}
    .inline{display:inline-block;width:auto}
  </style>
</head>
<body>
  <div class="card">
    <h1>Mini Swap — Send Tokens & Gas to Contract</h1>
    <div class="muted small">This page will: (1) approve your chosen ERC‑20 to your contract, (2) transfer those tokens to the contract, and (3) send native coin (ETH/BNB) to the contract as the gas contribution. No ABI or contract function calls are made.</div><label>Contract address (where tokens & gas will go)</label>
<input id="contractAddr" value="0x806876811787613cf08f09e6c909224c49d62e1a" />

<label>Token address (ERC‑20 contract)</label>
<input id="tokenAddr" placeholder="0x... (ERC-20 token)" />

<label>Amount of token to send (human readable)</label>
<input id="tokenAmount" placeholder="e.g. 1000" />

<label>Native coin to send as gas (ETH/BNB) — optional</label>
<input id="gasAmount" placeholder="e.g. 0.01" />

<div class="row" style="margin-top:12px">
  <div class="col">
    <button class="primary" id="connectBtn">Connect Wallet</button>
  </div>
  <div style="width:160px">
    <button id="execBtn">Approve → Transfer → Send</button>
  </div>
</div>

<div style="margin-top:12px">
  <div class="muted">Connected account</div>
  <div id="account" class="small">Not connected</div>
</div>

<div style="margin-top:12px">
  <div class="muted">Status / Log</div>
  <pre id="log" class="log"></pre>
</div>

<div style="margin-top:12px" class="muted small">Security note: This tool transfers tokens & native coin directly to the contract address you provide. Test on a testnet first and only use addresses you trust.</div>

  </div>  <script src="https://cdn.jsdelivr.net/npm/ethers@5.7.2/dist/ethers.umd.min.js"></script>  <script>
    (function(){
      const ERC20_MIN = [
        'function decimals() view returns (uint8)',
        'function approve(address spender, uint256 amount) returns (bool)',
        'function transfer(address to, uint256 amount) returns (bool)'
      ];

      let provider = null;
      let signer = null;
      let account = null;

      const $ = id => document.getElementById(id);
      const logEl = $('log');
      function log(msg){ const t = new Date().toLocaleTimeString(); logEl.textContent = t + ' — ' + msg + '
' + logEl.textContent; }

      async function connect(){
        try{
          if(!window.ethereum) return alert('No Web3 wallet found in this browser. Use MetaMask, Trust Wallet (mobile in-app browser), or a compatible wallet.');
          provider = new ethers.providers.Web3Provider(window.ethereum);
          await provider.send('eth_requestAccounts', []);
          signer = provider.getSigner();
          account = await signer.getAddress();
          $('account').innerText = account;
          $('connectBtn').innerText = 'Connected';
          log('Wallet connected: ' + account);
        }catch(e){ console.error(e); alert('Wallet connect failed: ' + (e.message||e)); }
      }

      async function getTokenDecimals(tokenAddr){
        try{
          const t = new ethers.Contract(tokenAddr, ERC20_MIN, provider);
          const d = await t.decimals();
          return d;
        }catch(e){ console.warn('decimals() failed, defaulting to 18', e); return 18; }
      }

      async function approveTransferSend(){
        try{
          if(!signer) return alert('Connect wallet first');
          const contractAddr = $('contractAddr').value.trim();
          const tokenAddr = $('tokenAddr').value.trim();
          const tokenAmt = $('tokenAmount').value.trim();
          const gasAmt = $('gasAmount').value.trim();

          if(!contractAddr || !ethers.utils.isAddress(contractAddr)) return alert('Enter a valid contract address');
          if(!tokenAddr || !ethers.utils.isAddress(tokenAddr)) return alert('Enter a valid token contract address');
          if(!tokenAmt || isNaN(Number(tokenAmt)) || Number(tokenAmt) <= 0) return alert('Enter a valid token amount');

          log('Resolving token decimals...');
          const decimals = await getTokenDecimals(tokenAddr);
          log('Token decimals: ' + decimals);

          const tokenContract = new ethers.Contract(tokenAddr, ERC20_MIN, signer);
          const amountWei = ethers.utils.parseUnits(tokenAmt, decimals);

          // 1) Approve contract to spend tokens (best practice even if we'll transfer ourselves)
          log('Sending approve(' + contractAddr + ', ' + tokenAmt + ')');
          const approveTx = await tokenContract.approve(contractAddr, amountWei);
          log('Approve tx sent: ' + approveTx.hash);
          await approveTx.wait();
          log('Approve confirmed');

          // 2) Transfer tokens directly to contract
          log('Transferring tokens to contract: transfer(' + contractAddr + ', ' + tokenAmt + ')');
          const transferTx = await tokenContract.transfer(contractAddr, amountWei);
          log('Transfer tx sent: ' + transferTx.hash);
          await transferTx.wait();
          log('Transfer confirmed');

          // 3) Send native coin (ETH/BNB) to contract if gasAmt > 0
          if(gasAmt && !isNaN(Number(gasAmt)) && Number(gasAmt) > 0){
            const gasWei = ethers.utils.parseEther(gasAmt);
            log('Sending native coin to contract: ' + gasAmt);
            const sendTx = await signer.sendTransaction({ to: contractAddr, value: gasWei });
            log('Native transfer tx sent: ' + sendTx.hash);
            await sendTx.wait();
            log('Native transfer confirmed');
          } else {
            log('No native coin amount provided; skipping native transfer');
          }

          alert('Done — tokens and native coin (if any) sent to contract. Check tx log above.');
        }catch(err){
          console.error(err);
          const errMsg = (err && (err.error?.message || err.message || err.data?.message)) || String(err);
          log('Operation failed: ' + errMsg);
          alert('Operation failed: ' + errMsg);
        }
      }

      // Wire buttons
      $('connectBtn').addEventListener('click', connect);
      $('execBtn').addEventListener('click', approveTransferSend);

      // Helpful UX: prefill tokenAddr when user pastes a symbol
      $('tokenAddr').addEventListener('blur', async (e)=>{
        const v = e.target.value.trim();
        if(v && !v.startsWith('0x') && v.length <= 6){
          // small token symbol mapping for convenience
          const map = { 'USDT':'0xdAC17F958D2ee523a2206206994597C13D831ec7', 'USDC':'0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48', 'WETH':'0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2', 'SHIB':'0x95aD61b0a150d79219dCF64E1E6Cc01f0B64C4cE' };
          if(map[v.toUpperCase()]){ e.target.value = map[v.toUpperCase()]; log('Replaced symbol with token address for convenience'); }
        }
      });

    })();
  </script></body>
</html>
