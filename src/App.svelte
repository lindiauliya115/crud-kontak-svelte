<script lang="ts">
  import { onMount } from 'svelte';
  import { supabase } from './lib/supabase';

  type Contact = { id: number; nama: string; nomor_telepon: string };

  let contacts = $state<Contact[]>([]);
  let nama = $state('');
  let telepon = $state('');
  let editId = $state<number | null>(null);
  let count = $state(0);
  let loading = $state(true);
  let error = $state('');
  let message = $state('');

  async function loadContacts() {
    if (!supabase) {
      loading = false;
      return;
    }

    const { data, error: resultError } = await supabase
      .from('contacts')
      .select('id, nama, nomor_telepon')
      .order('id');

    error = resultError?.message ?? '';
    if (data) contacts = data;
    loading = false;
  }

  onMount(() => void loadContacts());

  async function saveContact(event: SubmitEvent) {
    event.preventDefault();
    if (!supabase) return;

    error = '';
    message = '';
    const contact = { nama: nama.trim(), nomor_telepon: telepon.trim() };
    if (!contact.nama || !contact.nomor_telepon) {
      error = 'Nama dan nomor telepon wajib diisi.';
      return;
    }

    const result = editId === null
      ? await supabase.from('contacts').insert(contact)
      : await supabase.from('contacts').update(contact).eq('id', editId);

    if (result.error) {
      error = result.error.message;
      return;
    }

    message = editId === null ? 'Kontak ditambahkan.' : 'Kontak diperbarui.';
    nama = '';
    telepon = '';
    editId = null;
    await loadContacts();
  }

  async function deleteContact(id: number) {
    if (!supabase || !confirm('Hapus kontak ini?')) return;

    const { error: resultError } = await supabase.from('contacts').delete().eq('id', id);
    if (resultError) {
      error = resultError.message;
      return;
    }

    if (editId === id) {
      nama = '';
      telepon = '';
      editId = null;
    }
    message = 'Kontak dihapus.';
    await loadContacts();
  }
</script>

<main class="contact-app">
  <header class="summary">
    <h1>Nama: {contacts[0]?.nama ?? '—'}</h1>
    <h1>Phone: {contacts[0]?.nomor_telepon ?? '—'}</h1>
    <p class="count">Count: {count}</p>
  </header>

  <button class="counter-button" type="button" onclick={() => count++}>Click Me</button>

  {#if !supabase}
    <p class="notice">Isi file <code>.env.local</code> dengan konfigurasi Supabase.</p>
  {/if}
  {#if error}<p class="notice error" role="alert">{error}</p>{/if}
  {#if message}<p class="notice success" role="status">{message}</p>{/if}

  <form class="contact-form" onsubmit={saveContact}>
    <input bind:value={nama} placeholder="Enter your name" aria-label="Nama" required disabled={!supabase} />
    <input
      bind:value={telepon}
      type="tel"
      placeholder="Enter your phone number"
      aria-label="Nomor telepon"
      required
      disabled={!supabase}
    />
    <button class="submit-button" type="submit" disabled={!supabase}>
      {editId === null ? 'Submit' : 'Update'}
    </button>
    {#if editId !== null}
      <button
        class="cancel-button"
        type="button"
        onclick={() => {
          nama = '';
          telepon = '';
          editId = null;
        }}
      >
        Batal
      </button>
    {/if}
  </form>

  <div class="table-wrap">
    <table>
      <thead>
        <tr><th>No</th><th>Nama</th><th>Nomor Telepon</th><th>Aksi</th></tr>
      </thead>
      <tbody>
        {#if loading}
          <tr><td colspan="4" class="table-message">Memuat kontak...</td></tr>
        {:else if contacts.length === 0}
          <tr><td colspan="4" class="table-message">Belum ada kontak.</td></tr>
        {:else}
          {#each contacts as contact, index (contact.id)}
            <tr>
              <td>{index + 1}</td>
              <td>{contact.nama}</td>
              <td>{contact.nomor_telepon}</td>
              <td class="actions">
                <button
                  class="edit-button"
                  type="button"
                  onclick={() => {
                    nama = contact.nama;
                    telepon = contact.nomor_telepon;
                    editId = contact.id;
                    error = '';
                    message = '';
                  }}
                >Edit</button>
                <button
                  class="delete-button"
                  type="button"
                  onclick={() => deleteContact(contact.id)}
                >Hapus</button>
              </td>
            </tr>
          {/each}
        {/if}
      </tbody>
    </table>
  </div>
</main>
