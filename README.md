const API = '/api';
const storageKeys = {
  token: 'cloudvault-token',
  theme: 'cloudvault-theme'
};

const state = {
  token: localStorage.getItem(storageKeys.token) || '',
  user: null,
  files: [],
  activity: [],
  buckets: [],
  storage: { total: 0, used: 0, available: 0, fileCount: 0, folderCount: 0, byType: {} },
  theme: localStorage.getItem(storageKeys.theme) || 'dark',
  page: 'home',
  search: '',
  filter: 'all',
  selectedIds: [],
  currentFolderId: null,
  uploads: [],
  notifications: [],
  previewItem: null,
  helpTopics: [],
  authMode: 'login',
  authForm: { name: '', email: 'demo@cloudvault.app', password: 'demo123' },
  modal: null,
  contextMenu: null,
  loading: true
};

const els = {};

const formatBytes = (bytes = 0) => {
  if (!bytes) return '0 B';
  const units = ['B', 'KB', 'MB', 'GB', 'TB'];
  const index = Math.min(Math.floor(Math.log(bytes) / Math.log(1024)), units.length - 1);
  const value = bytes / (1024 ** index);
  return `${value.toFixed(value >= 10 || index === 0 ? 0 : 1)} ${units[index]}`;
};

const formatDate = (value) => {
  if (!value) return '—';
  return new Date(value).toLocaleString([], { dateStyle: 'medium', timeStyle: 'short' });
};

const fileKind = (name = '', mime = '') => {
  const ext = String(name).split('.').pop().toLowerCase();
  if (['png', 'jpg', 'jpeg', 'gif', 'webp', 'svg'].includes(ext)) return 'image';
  if (['mp4', 'webm', 'mov', 'avi', 'mkv'].includes(ext)) return 'video';
  if (['mp3', 'wav', 'ogg', 'm4a'].includes(ext)) return 'audio';
  if (['pdf'].includes(ext)) return 'document';
  if (['txt', 'json', 'csv', 'md', 'xml', 'css', 'js', 'html'].includes(ext)) return 'document';
  if (mime.startsWith('image/')) return 'image';
  if (mime.startsWith('video/')) return 'video';
  if (mime.startsWith('audio/')) return 'audio';
  return 'other';
};

const iconClass = (item) => {
  if (item.type === 'folder') return 'folder';
  return fileKind(item.name, item.mimeType || '');
};

const apiFetch = async (url, options = {}) => {
  const headers = { ...(options.headers || {}) };
  if (state.token) headers.Authorization = `Bearer ${state.token}`;
  const response = await fetch(url, { ...options, headers });
  if (!response.ok) {
    const text = await response.text();
    throw new Error(text || 'Request failed');
  }
  const contentType = response.headers.get('content-type') || '';
  if (contentType.includes('application/json')) return response.json();
  return response;
};

const showToast = (message, type = 'success') => {
  const id = Date.now() + Math.random();
  state.notifications.push({ id, message, type });
  render();
  setTimeout(() => {
    state.notifications = state.notifications.filter((n) => n.id !== id);
    render();
  }, 3000);
};

const loadSettings = async () => {
  if (!state.token) return;
  try {
    const settings = await apiFetch(`${API}/settings`);
    state.theme = settings.theme || state.theme;
    localStorage.setItem(storageKeys.theme, state.theme);
    applyTheme();
  } catch (error) {
    console.warn(error.message);
  }
};

const applyTheme = () => {
  document.body.dataset.theme = state.theme;
  localStorage.setItem(storageKeys.theme, state.theme);
};

const fetchAll = async () => {
  if (!state.token) return;
  try {
    const [files, buckets, storage, activity] = await Promise.all([
      apiFetch(`${API}/files?search=${encodeURIComponent(state.search)}&parentId=${state.currentFolderId ?? 'null'}`),
      apiFetch(`${API}/buckets`),
      apiFetch(`${API}/storage`),
      apiFetch(`${API}/activity`)
    ]);

    state.files = files || [];
    state.buckets = buckets || [];
    state.storage = storage || { total: 0, used: 0, available: 0, fileCount: 0, folderCount: 0, byType: {} };
    state.activity = activity || [];
  } catch (error) {
    showToast(error.message || 'Failed to refresh', 'error');
  }
};

const login = async (event) => {
  event.preventDefault();
  try {
    const payload = state.authMode === 'login'
      ? { email: state.authForm.email, password: state.authForm.password }
      : { name: state.authForm.name, email: state.authForm.email, password: state.authForm.password };

    const result = await apiFetch(`${API}/auth/${state.authMode === 'login' ? 'login' : 'signup'}`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(payload)
    });

    state.token = result.token;
    localStorage.setItem(storageKeys.token, result.token);
    state.user = result.user;
    state.loading = false;
    await fetchAll();
    await loadSettings();
    render();
    showToast('Welcome to CloudVault', 'success');
  } catch (error) {
    showToast(error.message || 'Auth failed', 'error');
  }
};

const logout = () => {
  state.token = '';
  state.user = null;
  state.files = [];
  state.activity = [];
  localStorage.removeItem(storageKeys.token);
  render();
};

const createFolder = async () => {
  const name = prompt('Folder name');
  if (!name) return;
  try {
    await apiFetch(`${API}/folders`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ name, parentId: state.currentFolderId })
    });
    await fetchAll();
    showToast('Folder created', 'success');
    render();
  } catch (error) {
    showToast(error.message || 'Could not create folder', 'error');
  }
};

const createBucket = async () => {
  const name = prompt('Bucket name');
  if (!name) return;
  try {
    await apiFetch(`${API}/buckets`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ name, description: 'New bucket' })
    });
    await fetchAll();
    showToast('Bucket created', 'success');
    render();
  } catch (error) {
    showToast(error.message || 'Could not create bucket', 'error');
  }
};

const initiateUpload = async (files) => {
  if (!files || !files.length) return;
  const formData = new FormData();
  for (const file of files) formData.append('files', file);
  if (state.currentFolderId) formData.append('parentId', state.currentFolderId);

  const items = files.map((file) => ({
    id: `${Date.now()}-${Math.random()}`,
    name: file.name,
    size: file.size,
    type: 'uploading'
  }));

  state.uploads = [...items, ...state.uploads];
  render();

  try {
    await apiFetch(`${API}/files/upload`, {
      method: 'POST',
      body: formData
    });
    await fetchAll();
    state.uploads = []; 
    render();
    showToast(`${files.length} upload${files.length > 1 ? 's' : ''} complete`, 'success');
  } catch (error) {
    showToast(error.message || 'Upload failed', 'error');
  }
};

const handleDownload = async (item) => {
  if (!item || item.type !== 'file') return;
  window.location.href = `${API}/files/${item.id}/download?token=${state.token}`;
};

const handleDelete = async (id, permanent = false) => {
  try {
    await apiFetch(`${API}/files/${id}/delete`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ permanent })
    });
    await fetchAll();
    state.selectedIds = state.selectedIds.filter((entry) => entry !== id);
    showToast(permanent ? 'Permanently deleted' : 'Moved to Trash', 'success');
    render();
  } catch (error) {
    showToast(error.message || 'Delete failed', 'error');
  }
};

const handleRestore = async (id) => {
  try {
    await apiFetch(`${API}/files/${id}/restore`, { method: 'POST' });
    await fetchAll();
    showToast('Restored', 'success');
    render();
  } catch (error) {
    showToast(error.message || 'Restore failed', 'error');
  }
};

const handleRename = async (id) => {
  const item = state.files.find((file) => file.id === id);
  const name = prompt('Rename item', item?.name || '');
  if (!name) return;
  try {
    await apiFetch(`${API}/files/${id}/rename`, {
      method: 'PATCH',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ name })
    });
    await fetchAll();
    showToast('Renamed', 'success');
  } catch (error) {
    showToast(error.message || 'Rename failed', 'error');
  }
};

const handleFavorite = async (id, favorite) => {
  try {
    await apiFetch(`${API}/files/${id}/favorite`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ favorite })
    });
    await fetchAll();
    render();
  } catch (error) {
    showToast(error.message || 'Favorite action failed', 'error');
  }
};

const handleShare = async (id) => {
  const item = state.files.find((file) => file.id === id);
  const choice = prompt('Sharing mode: private | link', item?.shareMode || 'private');
  if (!choice) return;
  const permission = prompt('Permission: view | download | edit', item?.permissions || 'view');
  try {
    await apiFetch(`${API}/files/${id}/share`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ shareMode: choice, permissions: permission })
    });
    await fetchAll();
    showToast('Sharing updated', 'success');
  } catch (error) {
    showToast(error.message || 'Share failed', 'error');
  }
};

const openItem = async (item) => {
  if (item.type === 'folder') {
    state.currentFolderId = item.id;
    state.page = 'my-files';
    await fetchAll();
    render();
    return;
  }

  state.previewItem = item;
  render();
};

const renderAuth = () => `
  <div class="auth-shell">
    <div class="auth-card">
      <div class="brand-row">
        <div class="brand-mark">C</div>
        <div>
          <div class="brand-name">CloudVault</div>
          <div class="brand-sub">Secure file storage</div>
        </div>
      </div>
      <div class="auth-tabs">
        <button class="${state.authMode === 'login' ? 'active' : ''}" data-auth-tab="login">Sign in</button>
        <button class="${state.authMode === 'signup' ? 'active' : ''}" data-auth-tab="signup">Sign up</button>
      </div>
      <form id="auth-form" class="auth-form">
        ${state.authMode === 'signup' ? `<label>Display name<input name="name" value="${state.authForm.name}" /></label>` : ''}
        <label>Email<input name="email" type="email" value="${state.authForm.email}" /></label>
        <label>Password<input name="password" type="password" value="${state.authForm.password}" /></label>
        <button class="primary-btn" type="submit">${state.authMode === 'login' ? 'Sign in' : 'Create account'}</button>
      </form>
      <div class="auth-hint">Demo login: demo@cloudvault.app / demo123</div>
    </div>
  </div>
`;

const renderPage = () => {
  const files = state.files.filter((item) => {
    if (item.status === 'trash') return false;
    if (state.currentFolderId !== null) return item.parentId === state.currentFolderId;
    return item.parentId === null;
  });

  const filteredFiles = files.filter((item) => {
    if (!state.search) return true;
    return `${item.name} ${item.type}`.toLowerCase().includes(state.search.toLowerCase());
  });

  const storageUsage = state.storage.used / (state.storage.total || 1) * 100;

  return `
    <div class="shell">
      <aside class="sidebar">
        <div class="brand-row">
          <div class="brand-mark">C</div>
          <div>
            <div class="brand-name">CloudVault</div>
            <div class="brand-sub">v1.0.0</div>
          </div>
        </div>
        <nav class="nav-list">
          ${['home', 'my-files', 'buckets', 'folders', 'storage', 'trash', 'settings'].map((page) => `
            <button class="nav-item ${state.page === page ? 'active' : ''}" data-page="${page}">${page === 'home' ? 'Home' : page === 'my-files' ? 'My Files' : page === 'buckets' ? 'Buckets' : page === 'folders' ? 'Folders' : page === 'storage' ? 'Storage' : page === 'trash' ? 'Trash' : 'Settings'}</button>
          `).join('')}
        </nav>
        <div class="sidebar-footer">
          <div class="user-box">
            <div class="user-avatar">${(state.user?.name || 'U').slice(0, 1).toUpperCase()}</div>
            <div>
              <strong>${state.user?.name || 'User'}</strong>
              <small>${state.user?.email || 'demo@cloudvault.app'}</small>
            </div>
          </div>
          <button class="secondary-btn" data-action="logout">Sign out</button>
          <div class="footer-meta">Help · v1.0.0</div>
        </div>
      </aside>

      <main class="main-panel">
        <header class="topbar">
          <div class="topbar-left">
            <input id="search-input" class="search-input" value="${state.search}" placeholder="Search files, folders, tags..." />
          </div>
          <div class="topbar-actions">
            <button class="secondary-btn" id="upload-files-btn">Upload Files</button>
            <button class="secondary-btn" id="upload-folder-btn">Upload Folder</button>
            <button class="primary-btn" data-action="new-folder">New Folder</button>
            <button class="primary-btn" data-action="new-bucket">New Bucket</button>
          </div>
        </header>

        ${state.uploads.length ? `<div class="queue">${state.uploads.map((upload) => `
          <div class="queue-item">
            <div class="queue-row"><strong>${upload.name}</strong><span>${formatBytes(upload.size)}</span></div>
            <div class="progress"><span style="width: 75%"></span></div>
            <div class="queue-row"><small>Uploading...</small><button class="mini-btn" data-cancel-upload="${upload.id}">Cancel</button></div>
          </div>
        `).join('')}</div>` : ''}

        ${state.page === 'home' ? `
          <section class="grid-two">
            <div class="card">
              <div class="card-head"><h3>Storage overview</h3><span>${formatBytes(state.storage.used)} used</span></div>
              <div class="stats-grid">
                <div><label>Total storage</label><strong>${formatBytes(state.storage.total)}</strong></div>
                <div><label>Used storage</label><strong>${formatBytes(state.storage.used)}</strong></div>
                <div><label>Available</label><strong>${formatBytes(state.storage.available)}</strong></div>
                <div><label>Files</label><strong>${state.storage.fileCount || 0}</strong></div>
                <div><label>Folders</label><strong>${state.storage.folderCount || 0}</strong></div>
              </div>
              <div class="progress-track"><span style="width: ${Math.min(storageUsage, 100)}%"></span></div>
            </div>
            <div class="card">
              <div class="card-head"><h3>Quick actions</h3></div>
              <div class="quick-grid">
                <button class="action-btn" id="quick-upload">Upload Files</button>
                <button class="action-btn" id="quick-upload-folder">Upload Folder</button>
                <button class="action-btn" data-action="new-folder">New Folder</button>
                <button class="action-btn" data-action="new-bucket">New Bucket</button>
              </div>
            </div>
            <div class="card">
              <div class="card-head"><h3>Recent files</h3></div>
              <div class="list-stack">
                ${(state.files || []).slice(0, 5).map((item) => `
                  <div class="list-item">
                    <span class="icon ${iconClass(item)}">${(item.name || 'F').charAt(0).toUpperCase()}</span>
                    <div class="meta"><strong>${item.name}</strong><small>${item.type === 'folder' ? 'Folder' : 'File'} · ${formatBytes(item.size || 0)}</small></div>
                    <button class="mini-btn" data-open-item="${item.id}">Open</button>
                  </div>
                `).join('') || '<div class="empty-text">No recent files</div>'}
              </div>
            </div>
            <div class="card">
              <div class="card-head"><h3>Favorites</h3></div>
              <div class="list-stack">
                ${(state.files || []).filter((item) => item.favorite).slice(0, 5).map((item) => `
                  <div class="list-item">
                    <span class="icon ${iconClass(item)}">${(item.name || 'F').charAt(0).toUpperCase()}</span>
                    <div class="meta"><strong>${item.name}</strong><small>${item.favorite ? 'Starred' : 'Normal'}</small></div>
                    <button class="mini-btn" data-open-item="${item.id}">Open</button>
                  </div>
                `).join('') || '<div class="empty-text">No favorites yet</div>'}
              </div>
            </div>
          </section>
        ` : ''}

        ${state.page === 'my-files' ? `
          <section class="card">
            <div class="card-head">
              <h3>My Files</h3>
              <div class="toolbar">
                <select id="filter-select">
                  <option value="all">All</option>
                  <option value="image">Images</option>
                  <option value="document">Documents</option>
                  <option value="video">Videos</option>
                  <option value="audio">Audio</option>
                </select>
              </div>
            </div>
            <div class="breadcrumb">Home / ${state.currentFolderId ? 'Folder' : 'Root'}</div>
            <div class="table-wrap">
              <table class="file-table">
                <thead>
                  <tr><th>Name</th><th>Type</th><th>Size</th><th>Modified</th><th>Actions</th></tr>
                </thead>
                <tbody>
                  ${(filteredFiles || []).map((item) => `
                    <tr class="${state.selectedIds.includes(item.id) ? 'selected' : ''}" data-row-id="${item.id}" data-context="${item.id}">
                      <td><div class="file-name"><span class="icon ${iconClass(item)}">${(item.name || 'F').charAt(0).toUpperCase()}</span> ${item.name}</div></td>
                      <td>${item.type === 'folder' ? 'Folder' : item.name.split('.').pop().toUpperCase() || 'FILE'}</td>
                      <td>${item.type === 'folder' ? '—' : formatBytes(item.size || 0)}</td>
                      <td>${formatDate(item.modifiedAt)}</td>
                      <td class="row-actions">
                        <button class="mini-btn" data-open-item="${item.id}">Open</button>
                        <button class="mini-btn" data-share-item="${item.id}">Share</button>
                        <button class="mini-btn" data-favorite-item="${item.id}">${item.favorite ? '★' : '☆'}</button>
                      </td>
                    </tr>
                  `).join('') || '<tr><td colspan="5"><div class="empty-text">No files yet</div></td></tr>'}
                </tbody>
              </table>
            </div>
          </section>
        ` : ''}

        ${state.page === 'buckets' ? `
          <section class="grid-two">
            ${(state.buckets || []).map((bucket) => `
              <div class="card bucket-card">
                <div class="card-head"><h3>${bucket.name}</h3><span>${bucket.access || 'private'}</span></div>
                <p>${bucket.description || 'Project bucket'}</p>
                <div class="bucket-stats">
                  <div><label>Created</label><strong>${formatDate(bucket.createdAt)}</strong></div>
                  <div><label>Files</label><strong>${(state.files.filter((f) => f.bucketId === bucket.id)).length}</strong></div>
                  <div><label>Storage</label><strong>${formatBytes((state.files.filter((f) => f.bucketId === bucket.id)).reduce((sum, file) => sum + Number(file.size || 0), 0))}</strong></div>
                </div>
              </div>
            `).join('') || '<div class="card">No buckets yet</div>'}
          </section>
        ` : ''}

        ${state.page === 'storage' ? `
          <section class="card">
            <div class="card-head"><h3>Storage analytics</h3></div>
            <div class="stats-grid">
              <div><label>Total storage</label><strong>${formatBytes(state.storage.total)}</strong></div>
              <div><label>Used storage</label><strong>${formatBytes(state.storage.used)}</strong></div>
              <div><label>Free storage</label><strong>${formatBytes(state.storage.available)}</strong></div>
            </div>
            <div class="usage-breakdown">
              ${Object.entries(state.storage.byType || {}).map(([key, value]) => `
                <div class="usage-row"><span>${key}</span><strong>${formatBytes(value)}</strong></div>
              `).join('')}
            </div>
          </section>
        ` : ''}

        ${state.page === 'trash' ? `
          <section class="card">
            <div class="card-head"><h3>Trash</h3></div>
            <div class="list-stack">
              ${(state.files || []).filter((item) => item.status === 'trash').map((item) => `
                <div class="list-item">
                  <div class="meta"><strong>${item.name}</strong><small>${formatDate(item.deletedAt)}</small></div>
                  <div class="row-actions">
                    <button class="mini-btn" data-restore-item="${item.id}">Restore</button>
                    <button class="mini-btn danger" data-delete-item="${item.id}">Delete</button>
                  </div>
                </div>
              `).join('') || '<div class="empty-text">Trash is empty</div>'}
            </div>
          </section>
        ` : ''}

        ${state.page === 'settings' ? `
          <section class="grid-two">
            <div class="card">
              <div class="card-head"><h3>Appearance</h3></div>
              <label class="field"><span>Theme</span><select id="theme-select"><option value="dark" ${state.theme === 'dark' ? 'selected' : ''}>Dark</option><option value="light" ${state.theme === 'light' ? 'selected' : ''}>Light</option></select></label>
              <label class="field"><span>Accent</span><input id="accent-input" type="color" value="#6d7bff" /></label>
            </div>
            <div class="card">
              <div class="card-head"><h3>Notifications</h3></div>
              <label class="checkbox"><input type="checkbox" checked /> Upload complete</label>
              <label class="checkbox"><input type="checkbox" checked /> Upload failed</label>
              <label class="checkbox"><input type="checkbox" checked /> Sharing notifications</label>
            </div>
          </section>
        ` : ''}
      </main>
    </div>
  `;
};

const render = () => {
  const app = document.getElementById('app');
  if (!state.token || !state.user) {
    app.innerHTML = renderAuth();
    return;
  }

  app.innerHTML = `
    ${renderPage()}
    ${state.contextMenu ? `<div class="context-menu" style="left:${state.contextMenu.x}px;top:${state.contextMenu.y}px">
      <button data-context-open="${state.contextMenu.id}">Open</button>
      <button data-context-preview="${state.contextMenu.id}">Preview</button>
      <button data-context-download="${state.contextMenu.id}">Download</button>
      <button data-context-share="${state.contextMenu.id}">Share</button>
      <button data-context-rename="${state.contextMenu.id}">Rename</button>
      <button data-context-delete="${state.contextMenu.id}">Delete</button>
    </div>` : ''}
    ${state.previewItem ? `
      <div class="modal-overlay">
        <div class="modal-card preview-card">
          <div class="card-head"><h3>${state.previewItem.name}</h3><button class="mini-btn" id="close-preview">Close</button></div>
          ${['png','jpg','jpeg','gif','webp','svg'].includes((state.previewItem.name || '').split('.').pop().toLowerCase()) ? `<img class="preview-image" src="${API}/files/${state.previewItem.id}/content?token=${state.token}" alt="${state.previewItem.name}" />` : ['txt','md','csv','json','js','css','html'].includes((state.previewItem.name || '').split('.').pop().toLowerCase()) ? `<pre class="preview-text">Loading…</pre>` : `<div class="empty-text">Preview unavailable <a href="${API}/files/${state.previewItem.id}/download?token=${state.token}">Download File</a></div>`}
        </div>
      </div>
    ` : ''}
    <div class="toast-stack">
      ${state.notifications.map((n) => `<div class="toast ${n.type}">${n.message}</div>`).join('')}
    </div>
  `;

  if (state.previewItem && ['txt','md','csv','json','js','css','html'].includes((state.previewItem.name || '').split('.').pop().toLowerCase())) {
    fetch(`${API}/files/${state.previewItem.id}/content?token=${state.token}`)
      .then((res) => res.text())
      .then((text) => {
        const previewText = document.querySelector('.preview-text');
        if (previewText) previewText.textContent = text;
      })
      .catch(() => {
        const previewText = document.querySelector('.preview-text');
        if (previewText) previewText.textContent = 'Preview unavailable';
      });
  }
};

const bindGlobalEvents = () => {
  document.body.addEventListener('click', async (event) => {
    const pageBtn = event.target.closest('[data-page]');
    if (pageBtn) {
      state.page = pageBtn.dataset.page;
      render();
      return;
    }

    const authTab = event.target.closest('[data-auth-tab]');
    if (authTab) {
      state.authMode = authTab.dataset.authTab;
      render();
      return;
    }

    if (event.target.closest('[data-action="new-folder"]')) {
      await createFolder();
      return;
    }
    if (event.target.closest('[data-action="new-bucket"]')) {
      await createBucket();
      return;
    }
    if (event.target.closest('[data-action="logout"]')) {
      logout();
      return;
    }
    if (event.target.closest('#quick-upload') || event.target.closest('#upload-files-btn')) {
      document.getElementById('hidden-upload-input').click();
      return;
    }
    if (event.target.closest('#quick-upload-folder') || event.target.closest('#upload-folder-btn')) {
      document.getElementById('hidden-folder-input').click();
      return;
    }
    if (event.target.closest('#close-preview')) {
      state.previewItem = null;
      render();
      return;
    }
    if (event.target.closest('[data-open-item]')) {
      const id = event.target.closest('[data-open-item]').dataset.openItem;
      const item = state.files.find((file) => file.id === id);
      if (item) await openItem(item);
      return;
    }
    if (event.target.closest('[data-share-item]')) {
      const id = event.target.closest('[data-share-item]').dataset.shareItem;
      await handleShare(id);
      return;
    }
    if (event.target.closest('[data-favorite-item]')) {
      const id = event.target.closest('[data-favorite-item]').dataset.favoriteItem;
      const item = state.files.find((file) => file.id === id);
      if (item) await handleFavorite(id, !item.favorite);
      return;
    }
    if (event.target.closest('[data-restore-item]')) {
      const id = event.target.closest('[data-restore-item]').dataset.restoreItem;
      await handleRestore(id);
      return;
    }
    if (event.target.closest('[data-delete-item]')) {
      const id = event.target.closest('[data-delete-item]').dataset.deleteItem;
      await handleDelete(id, true);
      return;
    }
    if (event.target.closest('[data-context-open]')) {
      const id = event.target.closest('[data-context-open]').dataset.contextOpen;
      const item = state.files.find((file) => file.id === id);
      if (item) await openItem(item);
      state.contextMenu = null;
      render();
      return;
    }
    if (event.target.closest('[data-context-download]')) {
      const id = event.target.closest('[data-context-download]').dataset.contextDownload;
      const item = state.files.find((file) => file.id === id);
      if (item) await handleDownload(item);
      state.contextMenu = null;
      render();
      return;
    }
    if (event.target.closest('[data-context-share]')) {
      const id = event.target.closest('[data-context-share]').dataset.contextShare;
      await handleShare(id);
      state.contextMenu = null;
      render();
      return;
    }
    if (event.target.closest('[data-context-rename]')) {
      const id = event.target.closest('[data-context-rename]').dataset.contextRename;
      await handleRename(id);
      state.contextMenu = null;
      render();
      return;
    }
    if (event.target.closest('[data-context-delete]')) {
      const id = event.target.closest('[data-context-delete]').dataset.contextDelete;
      await handleDelete(id);
      state.contextMenu = null;
      render();
      return;
    }
    if (event.target.closest('[data-context-preview]')) {
      const id = event.target.closest('[data-context-preview]').dataset.contextPreview;
      const item = state.files.find((file) => file.id === id);
      state.previewItem = item || null;
      state.contextMenu = null;
      render();
      return;
    }
    if (event.target.closest('tr[data-row-id]')) {
      const row = event.target.closest('tr[data-row-id]');
      const id = row.dataset.rowId;
      const current = row.classList.contains('selected');
      if (event.ctrlKey || event.metaKey) {
        state.selectedIds = current ? state.selectedIds.filter((entry) => entry !== id) : [...state.selectedIds, id];
      } else {
        state.selectedIds = [id];
      }
      render();
      return;
    }

    if (!event.target.closest('.context-menu')) {
      state.contextMenu = null;
      render();
    }
  });

  document.body.addEventListener('contextmenu', (event) => {
    const row = event.target.closest('tr[data-row-id]');
    if (!row) return;
    const id = row.dataset.rowId;
    event.preventDefault();
    state.contextMenu = { x: event.clientX, y: event.clientY, id };
    render();
  });

  document.body.addEventListener('input', (event) => {
    if (event.target.id === 'search-input') {
      state.search = event.target.value;
      render();
      return;
    }
    if (event.target.id === 'theme-select') {
      state.theme = event.target.value;
      applyTheme();
      return;
    }
  });

  document.body.addEventListener('change', async (event) => {
    if (event.target.id === 'filter-select') {
      state.filter = event.target.value;
      render();
      return;
    }
    if (event.target.id === 'theme-select') {
      state.theme = event.target.value;
      await apiFetch(`${API}/settings`, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ theme: state.theme })
      });
      applyTheme();
      render();
      return;
    }
    if (event.target.id === 'accent-input') {
      await apiFetch(`${API}/settings`, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ accentColor: event.target.value })
      });
      document.documentElement.style.setProperty('--accent', event.target.value);
      render();
    }
  });

  document.body.addEventListener('submit', async (event) => {
    if (event.target.id === 'auth-form') {
      event.preventDefault();
      const form = new FormData(event.target);
      state.authForm = {
        name: form.get('name') || '',
        email: form.get('email') || '',
        password: form.get('password') || ''
      };
      await login(event);
    }
  });

  document.addEventListener('keydown', (event) => {
    if ((event.ctrlKey || event.metaKey) && event.key.toLowerCase() === 'k') {
      event.preventDefault();
      const search = document.getElementById('search-input');
      search?.focus();
    }
    if (event.key === 'Escape') {
      state.contextMenu = null;
      state.previewItem = null;
      render();
    }
  });

  const hiddenUpload = document.createElement('input');
  hiddenUpload.type = 'file';
  hiddenUpload.multiple = true;
  hiddenUpload.id = 'hidden-upload-input';
  hiddenUpload.style.display = 'none';
  hiddenUpload.addEventListener('change', (event) => initiateUpload([...event.target.files]));
  document.body.appendChild(hiddenUpload);

  const hiddenFolder = document.createElement('input');
  hiddenFolder.type = 'file';
  hiddenFolder.multiple = true;
  hiddenFolder.webkitdirectory = true;
  hiddenFolder.id = 'hidden-folder-input';
  hiddenFolder.style.display = 'none';
  hiddenFolder.addEventListener('change', (event) => initiateUpload([...event.target.files]));
  document.body.appendChild(hiddenFolder);

  document.body.addEventListener('dragover', (event) => {
    event.preventDefault();
    document.body.classList.add('drag-active');
  });

  document.body.addEventListener('dragleave', () => document.body.classList.remove('drag-active'));
  document.body.addEventListener('drop', (event) => {
    event.preventDefault();
    document.body.classList.remove('drag-active');
    if (event.dataTransfer?.files?.length) initiateUpload([...event.dataTransfer.files]);
  });
};

const init = async () => {
  applyTheme();
  if (!state.token) {
    state.loading = false;
    render();
    bindGlobalEvents();
    return;
  }

  try {
    const me = await apiFetch(`${API}/auth/me`);
    state.user = me.user;
    await fetchAll();
    await loadSettings();
    state.loading = false;
    render();
  } catch (error) {
    localStorage.removeItem(storageKeys.token);
    state.token = '';
    state.user = null;
    state.loading = false;
    render();
  }

  bindGlobalEvents();
};

init();
