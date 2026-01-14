<script>
import { onMount } from 'svelte';
import { auth, onAuthStateChanged, db, collection, getDocs, query, orderBy } from '../../firebase.js';

let user = null;
let authLoading = true;
let transactions = [];

onMount(() => {
  onAuthStateChanged(auth, (u) => {
    user = u;
    authLoading = false;
    if (user) {
      loadTransactionsFromFirestore();
    }
  });
});

async function loadTransactionsFromFirestore() {
  if (!user) return;
  const txCol = collection(db, 'users', user.uid, 'transactions');
  const q = query(txCol, orderBy('date', 'desc'));
  const snapshot = await getDocs(q);
  transactions = snapshot.docs.map(doc => ({ id: doc.id, ...doc.data() }));
}

$: sumIncome = transactions.filter(t => t.type === 'income').reduce((a, t) => a + +t.amount, 0);
$: sumExpense = transactions.filter(t => t.type === 'expense').reduce((a, t) => a + +t.amount, 0);

</script>

{#if authLoading}
  <p>Loading...</p>
{:else if !user}
  <p>You must be signed in to view your dashboard.</p>
{:else}
  <main style="max-width: 560px; margin: 3rem auto;">
    <h2 style="margin-bottom:1.3em;">Dashboard</h2>
    <section style="margin-bottom:2.2rem;">
      <div style="display:flex; align-items:center; gap:1em; margin-bottom:1.2em;">
        <img src={user.photoURL} alt="avatar" style="height:2.3em; width:2.3em; border-radius:50%; border:1px solid #bbb;" />
        <span style="font-weight:500; font-size:1.14em;">{user.displayName}</span>
      </div>
      <div style="display: flex; gap: 2rem; flex-wrap: wrap; margin-bottom: 1.1em;">
        <div><b>Total Income:</b> <span style="color:#21a15a;">{sumIncome.toFixed(2)}</span></div>
        <div><b>Total Expenses:</b> <span style="color:#c43131;">{sumExpense.toFixed(2)}</span></div>
        <div><b>Balance:</b> <span style="color:#2176ae;">{(sumIncome - sumExpense).toFixed(2)}</span></div>
      </div>
    </section>
    <a href="/transactions"><button style="background:#2176ae;color:white;padding:.7em 1.6em;border:none;border-radius:8px;font-weight:600;">View Transactions</button></a>
  </main>
{/if}
