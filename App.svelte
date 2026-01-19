<script>
function formatEuro(val) {
  return Number(val).toLocaleString('en-US', { style: 'currency', currency: 'EUR', minimumFractionDigits: 2 });
}

function formatDayMonth(dateStr) {
  const d = new Date(dateStr)
  function ordinal(n){const s=["th","st","nd","rd"],v=n%100;return n+(s[(v-20)%10]||s[v]||s[0])}
return isNaN(d) ? dateStr : `${d.toLocaleString('en-US', { month: 'long' })} ${ordinal(d.getDate())}`;
}

// Core data structures for the finance app
let name = '';
let transactions = [];
const incomeCategories = ['Salary', 'Loan'];
const expenseCategories = ['Food', 'Transport', 'Shopping', 'Entertainment', 'Other'];
let theme = localStorage.getItem('financeTheme') || 'light';
$: document.body.style.background = theme === 'dark' ? '#181828' : 'white';
const STORAGE_KEY = 'financeData_v1';
import { onMount } from 'svelte';
import { auth, provider, signInWithPopup, signOut as fbSignOut, onAuthStateChanged, db, collection, addDoc, getDocs, query, orderBy, deleteDoc, doc as fbDoc } from './firebase.js';
let user = null;
let authLoading = true;
onMount(() => {
  onAuthStateChanged(auth, (u) => {
    user = u;
    authLoading = false;
    if (user) {
      loadTransactionsFromFirestore();
    } else {
      transactions = [];
    }
  });
});
async function googleSignIn() {
  try {
    await signInWithPopup(auth, provider);
  } catch (e) {
    alert('Sign-in failed: ' + e.message);
  }
}
function googleSignOut() {
  fbSignOut(auth);
}
async function loadTransactionsFromFirestore() {
  if (!user) return;
  const txCol = collection(db, 'users', user.uid, 'transactions');
  const q = query(txCol, orderBy('date', 'desc'));
  const snapshot = await getDocs(q);
  transactions = snapshot.docs.map(doc => ({ id: doc.id, ...doc.data() }));
}
async function addIncome() {
  const tx = {
    type: 'income',
    amount: +incAmount,
    category: incCategory,
    date: incDate,
    note: incNote.trim(),
  };
  if (user) {
    const txCol = collection(db, 'users', user.uid, 'transactions');
    await addDoc(txCol, tx);
    await loadTransactionsFromFirestore();
  }
  incAmount = '';
  incNote = '';
  incCategory = incomeCategories[0];
  incDate = (new Date()).toISOString().slice(0,10);
  uiSection = 'home';
}
async function addExpense() {
  const tx = {
    type: 'expense',
    amount: +expAmount,
    category: expCategory,
    date: expDate,
    note: expNote.trim(),
  };
  if (user) {
    const txCol = collection(db, 'users', user.uid, 'transactions');
    await addDoc(txCol, tx);
    await loadTransactionsFromFirestore();
  }
  expAmount = '';
  expNote = '';
  expCategory = expenseCategories[0];
  expDate = (new Date()).toISOString().slice(0,10);
  uiSection = 'home';
}
async function deleteTransaction(id) {
  if (!user) return;
  const txRef = fbDoc(db, 'users', user.uid, 'transactions', id);
  await deleteDoc(txRef);
  await loadTransactionsFromFirestore();
}
function uuidv4() {
  return ([1e7]+-1e3+-4e3+-8e3+-1e11).replace(/[018]/g, c=>(c^window.crypto.getRandomValues(new Uint8Array(1))[0]&15>>c/4).toString(16));
}
let uiSection = 'home';
let incCategory = incomeCategories[0];
let incAmount = '';
let incDate = (new Date()).toISOString().slice(0,10);
let incNote = '';
let expCategory = expenseCategories[0];
let expAmount = '';
let expDate = (new Date()).toISOString().slice(0,10);
let expNote = '';
$: sumIncome = transactions.filter(t=>t.type=='income').reduce((a,t)=>a+ +t.amount,0);
$: sumExpense = transactions.filter(t=>t.type=='expense').reduce((a,t)=>a+ +t.amount,0);
</script>

<div style="display:flex; justify-content: flex-end; gap:1em; margin-bottom:.7em; align-items:center;">
  <button on:click={() => {
    theme = theme === 'light' ? 'dark' : 'light';
    localStorage.setItem('financeTheme', theme);
  }}>
    {theme === 'light' ? '🌙 Dark mode' : '☀️ Light mode'}
  </button>
</div>
{#if authLoading}
  <span>Loading user...</span>
{:else if !user}
  <section style="max-width: 520px; margin: 4rem auto; padding: 2.2rem 2rem; border-radius: 16px; background: var(--welcome-bg,#f8fafc); box-shadow: 0 4px 24px 2px #dbdbe544; text-align: center; font-family: 'Roboto', sans-serif;">
    <h1 style="font-weight: 600; font-size: 2.2em; color: #334e7e; margin-bottom: 0.8em;">Welcome to Money Tracker</h1>
    <div style="font-size:1.17em; color:#405368; margin-bottom:2em; line-height:1.7;">
      <p>
        Track your income and expenses with ease. Get clear insights into your personal or business finances.<br>
        Secure, cloud-synced, and private: your data belongs to you.<br>
      </p>
      <div style="margin: 1.3em 0 2.2em 0;">
        <b>Features you'll love:</b>
        <ul style="text-align:left; display:inline-block; color:#2d445e;">
          <li style="margin-bottom: .9em;"><b>✔️ Fast Add:</b> One-tap for recurring transactions</li>
          <li style="margin-bottom: .9em;"><b>✔️ Visual Insights:</b> Instant income/expense breakdowns</li>
          <li style="margin-bottom: .9em;"><b>✔️ Secure Sync:</b> Google-authenticated and private</li>
          <li style="margin-bottom: .9em;"><b>✔️ Dark Mode & Accessibility:</b> Comfortable on every device</li>
        </ul>
      </div>
      <span style="color: #4985e7; font-weight: 500;">Start now and take control of your spending habits!</span>
    </div>
    <button on:click={googleSignIn} style="background: linear-gradient(90deg,#4285f4,#1967d2); color: #fff; padding: 0.9em 2.2em; border: none; border-radius: 8px; font-size: 1.25em; font-weight: 600; box-shadow: 0 1.5px 7px rgba(72,100,195,0.13); letter-spacing: .01em; transition:background 0.2s; cursor:pointer;">Sign in with Google</button>
  </section>
{:else}
  <div style="display:inline-flex; align-items:center; gap:0.7em; margin-bottom:.7em; justify-content:flex-end; width:100%">
    <img src={user.photoURL} alt="avatar" style="height:2em; width:2em; border-radius:50%; border:1px solid #bbb;" />
    <span style="font-size:.97em;">{user.displayName}</span>
    <button on:click={googleSignOut}>Sign out</button>
  </div>
{/if}
{#if user}
<main class={theme} style="font-family: sans-serif; padding: 2rem; width: 100%; max-width: 100vw; margin-left: 20px; margin-right: 20px; box-sizing: border-box;">
  {#if uiSection === 'home'}
    <!-- Home dashboard -->
    <section>
      <h2 style="margin-bottom: 1em;">Current Status</h2>
      <div class="card-row">
  <div class="status-card income-card">
    <span class="card-icon" style="background:#43ce80;"><svg width="18" height="18" viewBox="0 0 24 24" fill="none" xmlns="http://www.w3.org/2000/svg"><path d="M12 21v-6m0 0-3 3m3-3 3 3M12 3v12" stroke="#fff" stroke-width="1.7" stroke-linecap="round" stroke-linejoin="round"/></svg></span>
    <span class="status-label">Total Income</span>
    <span class="status-value">{formatEuro(sumIncome)}</span>
  </div>
  <div class="status-card expense-card">
    <span class="card-icon" style="background:#e65447;"><svg width="18" height="18" viewBox="0 0 24 24" fill="none" xmlns="http://www.w3.org/2000/svg"><path d="M12 3v6m0 0-3-3m3 3 3-3M12 21V9" stroke="#fff" stroke-width="1.7" stroke-linecap="round" stroke-linejoin="round"/></svg></span>
    <span class="status-label">Total Expenses</span>
    <span class="status-value">{formatEuro(sumExpense)}</span>
  </div>
  <div class="status-card balance-card">
    <span class="card-icon" style="background:#1967d2;"><svg width="18" height="18" viewBox="0 0 24 24" fill="none" xmlns="http://www.w3.org/2000/svg"><circle cx="12" cy="12" r="10" stroke="#fff" stroke-width="1.7"/><path d="M8 15c0-.928 1.791-2 4-2s4 1.072 4 2m-6-4v-1a2 2 0 0 1 4 0v1" stroke="#fff" stroke-width="1.7" stroke-linecap="round" stroke-linejoin="round"/></svg></span>
    <span class="status-label">Balance</span>
    <span class="status-value">{formatEuro(sumIncome - sumExpense)}</span>
  </div>
</div>
      <div style="display: flex; gap: 2em;">
        <button on:click={() => uiSection = 'add-income'}>+ Add Income</button>
        <button on:click={() => uiSection = 'add-expense'}>- Add Expense</button>
      </div>
    </section>
  {/if}
  {#if uiSection === 'add-income'}
    <section>
      <h2>Add Income</h2>
      <form on:submit|preventDefault={addIncome} style="display: flex; flex-wrap: wrap; gap: 0.5rem; align-items: center;">
        <span>Type: <b>Income</b></span>
        <select bind:value={incCategory} required>
          {#each incomeCategories as cat}<option value={cat}>{cat}</option>{/each}
        </select>
        <input type="number" min="0.01" step="0.01" bind:value={incAmount} placeholder="Amount" required style="width:8em;" />
        <input type="date" bind:value={incDate} required style="width:9em;" />
        <input bind:value={incNote} placeholder="Note (optional)" style="flex:1;" />
        <button type="submit">Add</button>
        <button type="button" on:click={() => uiSection='home'} style="background:transparent;color:#444;border:1px solid #bbb;">Cancel</button>
      </form>
    </section>
  {/if}
  {#if uiSection === 'add-expense'}
    <section>
      <h2>Add Expense</h2>
      <form on:submit|preventDefault={addExpense} style="display: flex; flex-wrap: wrap; gap: 0.5rem; align-items: center;">
        <span>Type: <b>Expense</b></span>
        <select bind:value={expCategory} required>
          {#each expenseCategories as cat}<option value={cat}>{cat}</option>{/each}
        </select>
        <input type="number" min="0.01" step="0.01" bind:value={expAmount} placeholder="Amount" required style="width:8em;" />
        <input type="date" bind:value={expDate} required style="width:9em;" />
        <input bind:value={expNote} placeholder="Note (optional)" style="flex:1;" />
        <button type="submit">Add</button>
        <button type="button" on:click={() => uiSection='home'} style="background:transparent;color:#444;border:1px solid #bbb;">Cancel</button>
      </form>
    </section>
  {/if}
  <!-- Transactions List -->
  <section style="margin: 2rem 0;">
    <h2>Transactions</h2>
    {#if transactions.length === 0}
      <p style="color: #888;">No transactions yet.</p>
    {:else}
      <table style="width:100%; background: white; border-radius:8px; box-shadow:0 2px 7px #eee;">
        <thead><tr style="text-align:left;">
          <th>Date</th><th>Type</th><th>Amount</th><th>Category</th><th>Note</th>
        </tr></thead>
        <tbody>
          {#each transactions.slice().sort((a, b) => b.date.localeCompare(a.date)) as t (t.id)}
            <tr>
              <td>{formatDayMonth(t.date)}</td>
              <td><span class="badge {t.type}-badge">{t.type === 'income' ? 'Income' : 'Expense'}</span></td>
              <td class={t.type === 'income' ? 'income-amount' : 'expense-amount'}>{t.type=='income' ? '+' : '-'}{formatEuro(t.amount)}</td>
              <td>{t.category}</td>
              <td>{t.note}</td>
              <td><button class="delete-btn" on:click={() => deleteTransaction(t.id)} data-tooltip="Delete">🗑️</button></td>
            </tr>
          {/each}
        </tbody>
      </table>
    {/if}
  </section>
  <!-- Totals and Analytics -->
  <section style="margin:2rem 0; padding:1rem; border-radius:8px;">
    <h2>Totals & Analytics</h2>
    <div>Total Income: <b class='total income'>{sumIncome.toFixed(2)}</b></div>
    <div>Total Expenses: <b class='total expenses'>{sumExpense.toFixed(2)}</b></div>
    <div>Balance: <b class='total balance'>{(sumIncome - sumExpense).toFixed(2)}</b></div>
  </section>
</main>
{/if}
<style>
  .delete-btn { background: none; border: none; font-size: 1.1em; cursor: pointer; transition: background 0.17s; position: relative; }
  .delete-btn:hover, .delete-btn:focus { background: #eee; color: #c92a2a; }
  /* Tooltip Styles */
  .delete-btn::before {
    content: attr(data-tooltip);
    position: absolute;
    top: 50%;
    right: 100%;
    transform: translateY(-50%) translateX(-8px);
    background: #333;
    color: #fff;
    padding: 0.4em 0.8em;
    border-radius: 6px;
    font-size: 0.75em;
    font-weight: 500;
    white-space: nowrap;
    opacity: 0;
    pointer-events: none;
    transition: all 0.2s ease;
    box-shadow: 0 2px 10px rgba(0,0,0,0.2);
    margin-right: 8px;
    z-index: 10;
  }
  .delete-btn:hover::before {
    opacity: 1;
    transform: translateY(-50%) translateX(0);
  }
  main.dark .delete-btn:hover, main.dark .delete-btn:focus { background: #262338; color: #fb7185; }
  main.dark .delete-btn::before { background: #eee; color: #222; }
  @import url('https://fonts.googleapis.com/css2?family=Roboto:wght@400;500&display=swap');
  body, main { font-family: 'Roboto', Arial, Helvetica, sans-serif; }
  main { padding: 2rem; max-width: 560px; background: #fff; color: #222; border-radius: 12px; box-shadow: 0 3px 21px 3px rgba(60,64,67,0.08), 0 1.5px 3px 0 rgba(60,64,67,0.16); margin: 2.5rem auto 0 auto; }
  h1, h2 { font-weight: 500; letter-spacing: 0.02em; }
  section { background: #fafbfc; border-radius: 10px; box-shadow: 0 1px 3px 0 rgba(60,64,67,0.07); padding: 1.3rem 1rem; margin-bottom: 1.3rem; }
  label, input, select, button { font-family: inherit; font-size: 1rem; }
  input, select { border: 1.5px solid #dadce0; border-radius: 6px; padding: 0.55em 1em; margin-right: 0.5em; outline: none; background: #fafafa; box-shadow: none; transition: border-color 0.15s; }
  input:focus, select:focus { border-color: #4285f4; background: #fff; }
  button { background: #4285f4; color: #fff; border: none; border-radius: 6px; padding: 0.65em 1.4em; font-weight: 500; box-shadow: 0 2px 6px 0 rgba(60,64,67,0.12); cursor: pointer; letter-spacing: 0.011em; transition: background 0.17s, box-shadow 0.17s; margin-left: 0.7em; }
  button:hover { background: #1967d2; box-shadow: 0 2px 10px 0 rgba(60,64,67,0.17); }
  table {
  width: 100%;
  border-radius: 13px;
  border-collapse: separate;
  border-spacing: 0;
  box-shadow: 0 2px 14px 0 #10264228;
  overflow: hidden;
  margin: 1.5em 0 1em 0;
  background: #fff;
}
thead tr th {
  background: linear-gradient(90deg, #4285f4 0%, #1967d2 100%);
  color: #fff;
  font-weight: 600;
  letter-spacing: 0.04em;
  font-size: 1.03em;
  padding: 0.36em 0.75em 0.33em 0.75em;
  border: none;
  position: sticky;
  top: 0;
  z-index: 1;
  box-shadow: 0 2px 12px #679df519;
}
tbody tr:nth-child(even) {
  background: #f8fafd;
}
tbody tr:hover {
  background: #e9f2ff;
}
td {
  padding: 0.19em 0.75em;
  text-align: left;
  border: none;
}
tbody tr td {
  border-bottom: 1px solid #e5ebf2;
}
tbody tr:last-child td {
  border-bottom: none;
}
tbody tr {
  transition: background 0.18s;
  height: 2.17em;
  font-size: 1.06em;
}

  th, td { text-align: left; padding: 0.6em 0.75em; }
  th { background: #f4f6fa; font-weight: 500; color: #5f6368; letter-spacing: 0.015em; }
  tr:not(:last-child) td { border-bottom: 1px solid #e0e3e7; }
  section:last-of-type div { margin-bottom: 0.5rem; }

.card-row {
  display: flex;
  gap: 1.1rem;
  flex-wrap: wrap;
  margin-bottom: 1.7em;
}
.status-card {
  flex: 1 0 120px;
  min-width: 130px;
  background: #fff;
  border-radius: 13px;
  box-shadow: 0 2.5px 15px -3px #638ceb12, 0 0.5px 2px #ddebf833;
  padding: 1.2em 1.1em 0.95em 1.1em;
  display: flex;
  flex-direction: column;
  align-items: flex-start;
  justify-content: center;
}
.card-icon {
  margin-left: auto;
  margin-bottom: 0.52em;
  display: block;
  border-radius: 50%;
  padding: 5px;
  box-shadow: 0 1px 8px #95bafa25;
}
.status-label {
  font-size: 1.01em;
  color: #6a7c96;
  letter-spacing: .01em;
  margin-bottom: 0.43em;
  font-weight: 500;
}
.status-value {
  font-size: 1.28em;
  letter-spacing: .01em;
  font-weight: 700;
  margin-top: 0.15em;
}
.income-card {
  border-left: 6px solid #43ce80;
}
.expense-card {
  border-left: 6px solid #e65447;
}
.balance-card {
  border-left: 6px solid #1967d2;
}

  main.dark { background: #1c1b1f; color: #ece6f0; }
  main.dark section { background: #232029; box-shadow: 0 2px 8px rgba(0,0,0,0.24), 0 1.5px 4px rgba(0,0,0,0.12), 0 1.5px 5px 0 rgba(0,0,0,0.19), 0 1px 10px 0 rgba(60,64,67,0.12); }
  main.dark table { background: #22222a; }
  main.dark th { background: #262338; color: #bfc9db; }
  main.dark td { color: #f0eff4; }
  main.dark input, main.dark select { background: #23212A; color: #f0eff4; border-color: #49454f; box-shadow: none; transition: border-color 0.18s, box-shadow 0.18s; }
  main.dark input:focus, main.dark select:focus { background: #23242f; border-color: #82b1ff; box-shadow: 0 0 0 2px #1e88e5cc; }
  main.dark input::placeholder, main.dark select::placeholder { color: #b1b1c1; opacity: 1; }
  main.dark button { background: #1e88e5; color: #f0eff4; border-radius: 7px; border: none; box-shadow: 0 2px 6px 0 rgba(30,136,229,0.23); transition: background 0.18s, box-shadow 0.14s, color 0.18s; }
  main.dark button:hover, main.dark button:active { background: #90caf9; color: #181828; box-shadow: 0 4px 14px 0 rgba(30,136,229,0.28); }
  main.dark button:focus-visible { outline: 2px solid #bb86fc; outline-offset: 2px; }
  ::selection { background: #afe1fb; }
  main.dark ::selection { background: #225391; }
  @media (max-width: 600px) { main { max-width: 100vw; padding: 0.6rem; } table, section { font-size: 0.97em; } section { padding: 0.74rem 0.4rem; margin-bottom: 1em; } }
  .total.income { color: #219653; }
  .total.expenses { color: #c43131; }
  .total.balance { color: #1e88e5; }
  main.dark .total.income { color: #4ade80; }
  main.dark .total.expenses { color: #fb7185; }
  main.dark .total.balance { color: #90caf9; }
  .income-amount { color: #219653; font-weight: 500; }
  .expense-amount { color: #c43131; font-weight: 500; }
  main.dark .income-amount { color: #4ade80; }
  main.dark .expense-amount { color: #fb7185; }
.badge {
  display: inline-block;
  font-size: .97em;
  font-weight: 500;
  padding: 0.19em 1.1em 0.17em 1.1em;
  border-radius: 14px;
  letter-spacing: .01em;
  text-transform: capitalize;
  background: #e3eafc;
  color: #2761c3;
  box-shadow: 0 1px 4px 0 #b6d3f880;
  border: none;
  vertical-align: middle;
}
.income-badge {
  background: #e6f4ea;
  color: #17633c;
  box-shadow: 0 1px 7px 0 #4ade807c;
}
.expense-badge {
  background: #fce8e6;
  color: #a80e18;
  box-shadow: 0 1px 7px 0 #fb71856c;
}
</style>
