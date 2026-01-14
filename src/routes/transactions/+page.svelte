<script>

</script>

<style>
.badge-category {
}

.trash-btn {
  background: none;
  border: none;
  color: #c43131;
  cursor: pointer;
  padding: 0.3em 0.5em;
  border-radius: 50%;
  transition: background 0.18s;
  position: relative;
}
.trash-btn:hover, .trash-btn:focus {
  background: #fff3f3;
  color: #a50e0e;
  outline: none;
}
.modal-backdrop {
  position: fixed;
  top: 0; left: 0; right: 0; bottom: 0;
  background: rgba(30,37,50,0.27);
  z-index: 1111;
}
.modal-confirm {
  position: fixed;
  top: 44vh; left: 50%;
  transform: translate(-50%, -30%);
  z-index: 1120;
  background: #fff;
  padding: 1.35em 2.4em;
  border-radius: 16px;
  box-shadow: 0 2px 21px #aaa5;
  min-width: 300px;
  text-align: center;
  font-size: 1.05em;
}
.modal-actions {
  margin-top: 1.3em;
  display: flex; gap: 1.2em; justify-content: center;
}
.modal-btn {
  min-width: 64px;
  padding: 0.55em 1.4em;
  border: none;
  border-radius: 8px;
  font-weight: 600;
  font-size: 1em;
  cursor: pointer;
  transition: background 0.16s;
}
.modal-btn.cancel {
  background: #edeef7;
  color: #565674;
}
.modal-btn.cancel:hover { background: #dcdcf2; }
.modal-btn.delete {
  background: #ffebee;
  color: #c43131;
}
.modal-btn.delete:hover {
  background: #ffcdd2;
}
.undo-bar {
  z-index: 1130;
  position: fixed; bottom: 26px; left: 50%; transform: translateX(-50%);
  display: flex; align-items: center; gap: 1.2em;
  background: #fff;
  color: #1a1647;
  box-shadow: 0 2px 10px #9988;
  border-radius: 8px;
  padding: 0.92em 2.3em;
  font-size: 1.08em;
  animation: fadeIn 0.19s;
}
.undo-btn {
  background: none; border: none;
  color: #1967d2; font-weight: 600;
  cursor: pointer;
  font-size: 1em;
  border-radius: 5px;
  padding: 0.1em 0.6em;
  transition: background 0.13s;
}
.undo-btn:hover { background: #e3eafc; }

.trash-btn svg {
  width: 1.2em;
  height: 1.2em;
  vertical-align: middle;
}
.trash-tooltip {
  visibility: hidden;
  background: #2d2d36;
  color: #fff;
  text-align: center;
  border-radius: 5px;
  padding: 0.33em 0.7em;
  position: absolute;
  z-index: 1;
  bottom: 140%;
  left: 50%;
  transform: translateX(-50%);
  font-size: 0.96em;
  opacity: 0;
  pointer-events: none;
  transition: opacity 0.18s;
}
.trash-btn:hover .trash-tooltip, .trash-btn:focus .trash-tooltip {
  visibility: visible;
  opacity: 1;
}

  background: #e3eafc;
  color: #1967d2;
  border-radius: 12px;
  padding: 0.15em 0.85em;
  font-size: 0.98em;
  display: inline-block;
  font-weight: 500;
  letter-spacing: 0.01em;
  box-shadow: 0 1.5px 6px #b2b7c080;
}
</style>

import { onMount } from 'svelte';
import { auth, onAuthStateChanged, db, collection, getDocs, query, orderBy, deleteDoc, doc as fbDoc } from '../../firebase.js';

let user = null;
let authLoading = true;
let transactions = [];
let showConfirm = false;
let confirmId = null;


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

function promptDelete(id, tx) {
  confirmId = id;
  recentlyDeletedData = tx;
  showConfirm = true;
}

async function doDelete(id) {
  if (!user) return;
  const txRef = fbDoc(db, 'users', user.uid, 'transactions', id);
  await deleteDoc(txRef);
  recentlyDeleted = id;
  showConfirm = false;
  confirmId = null;
  await loadTransactionsFromFirestore();
  // Set up undo timeout
  clearTimeout(undoTimeout);
  undoTimeout = setTimeout(() => { recentlyDeleted = null; recentlyDeletedData = null; }, 6000);
}

function cancelDelete() {
  showConfirm = false;
  confirmId = null;
}

async function undoDelete() {
  if (!user || !recentlyDeletedData) return;
  // Re-add transaction using firestore add method
  const txCol = collection(db, 'users', user.uid, 'transactions');
  const tx = Object.assign({}, recentlyDeletedData);
  delete tx.id;
  await txCol._add(tx); // If using Firestore SDK v9, use addDoc()
  recentlyDeleted = null;
  recentlyDeletedData = null;
  await loadTransactionsFromFirestore();
  clearTimeout(undoTimeout);
}


{#if showConfirm}
  <div class="modal-backdrop"></div>
  <div class="modal-confirm">
    <div>Are you sure you want to delete this transaction?</div>
    <div class="modal-actions">
      <button class="modal-btn cancel" on:click={cancelDelete}>Cancel</button>
      <button class="modal-btn delete" on:click={() => doDelete(confirmId)}>Delete</button>
    </div>
  </div>
{/if}


  <div class="modal-backdrop"></div>
  <div class="modal-confirm">
    <div>Are you sure you want to delete this transaction?</div>
    <div class="modal-actions">
      <button class="modal-btn cancel" on:click={cancelDelete}>Cancel</button>
      <button class="modal-btn delete" on:click={() => doDelete(confirmId)}>Delete</button>
    </div>
  </div>
{/if}

{#if recentlyDeleted}
  <div class="undo-bar">
    <span>Transaction deleted.</span>
    <button on:click={undoDelete} class="undo-btn">Undo</button>
  </div>
{/if}

{#if authLoading}
  <p>Loading...</p>
{:else if !user}
  <p>You must be signed in to view your transactions.</p>
{:else}
  <main style="max-width: 700px; margin: 3.5rem auto;">
    <h2 style="margin-bottom:1.4em;">Transactions</h2>
    <section>
      {#if transactions.length === 0}
        <p>No transactions yet.</p>
      {:else}
        <table style="width:100%; background: white; border-radius:8px; box-shadow:0 2px 7px #eee;">
          <thead>
            <tr>
              <th>Date</th><th>Type</th><th>Amount</th><th>Category</th><th>Note</th><th>Delete</th>
            </tr>
          </thead>
          <tbody>
            {#each transactions as t (t.id)}
              <tr>
                <td>{t.date}</td>
                <td>{t.type}</td>
                <td style="color:{t.type==='income'?'#21a15a':'#c43131'};font-weight:500;">{t.type==='income'?'+':'-'}{(+t.amount).toFixed(2)}</td>
                <td><span class="badge-category">{t.category}</span></td>
                <td>{t.note}</td>
                <td>
                <button class="trash-btn" on:click={() => promptDelete(t.id, t)} aria-label="Delete transaction">
                  <svg viewBox="0 0 20 20" fill="currentColor" aria-hidden="true">
                    <path d="M7.2 3.5h5.6a.7.7 0 0 1 .7.7v.6H18a.8.8 0 0 1 0 1.6h-1.1l-.7 10.4A2.27 2.27 0 0 1 13.93 19H6.08A2.27 2.27 0 0 1 3.8 16.8L3.1 6.4H2A.8.8 0 0 1 2 4.8h4.5v-.6a.7.7 0 0 1 .7-.7zM4.7 16.7c0 .37.3.7.68.7h7.22a.7.7 0 0 0 .7-.7L14 6.4H4l.7 10.3zm3.8-7.2v5.4a.7.7 0 1 0 1.4 0V9.5a.7.7 0 1 0-1.4 0zm-2.1.7a.7.7 0 0 1 1.4 0v4.7a.7.7 0 1 1-1.4 0V10.2zm5.6 0v4.7a.7.7 0 1 1-1.4 0v-4.7a.7.7 0 1 1 1.4 0z"/>
                  </svg>
                  <span class="trash-tooltip">Delete transaction</span>
                </button>
              </td>
              </tr>
            {/each}
          </tbody>
        </table>
      {/if}
    </section>
    <a href="/dashboard"><button style="margin:2em 0 0 0; background:#2176ae;color:white;padding:.7em 1.6em;border:none;border-radius:8px;font-weight:600;">Back to Dashboard</button></a>
  </main>
{/if}
