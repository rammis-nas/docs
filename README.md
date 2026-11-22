# GitHub Docs <!-- omit in toc -->
'{/<nesru.dev>}' <!-- ==========================================================
     NESRU MICHAEL APPEND-ONLY ENHANCEMENTS (BEFORE </body>)
     ========================================================== -->
<div id="nm-append-notif-modal" class="modal-backdrop hidden" onclick="if(event.target===this) nmCloseNotifModal()">
  <div class="modal-card">
    <div class="modal-header">
      <h3>🔔 Beeksisa (Notifications)</h3>
      <div style="display:flex;gap:6px;">
        <button type="button" class="btn btn-outline btn-sm" onclick="nmMarkAllRead()">✓ Dubbise</button>
        <button type="button" class="btn btn-outline btn-sm" onclick="nmCloseNotifModal()">✕ Cufi</button>
      </div>
    </div>
    <div id="nm-append-notif-list" style="font-size:14px;"></div>
  </div>
</div>

<script>
(function () {
  const NOTIF_KEY = "nesru_notifications_v2";
  let nmNotifs = JSON.parse(localStorage.getItem(NOTIF_KEY) || "[]");
  let nmSelectedFile = null;

  function nmEsc(str) {
    if (str === null || str === undefined) return "";
    return String(str)
      .replace(/&/g, "&amp;")
      .replace(/</g, "&lt;")
      .replace(/>/g, "&gt;")
      .replace(/"/g, "&quot;")
      .replace(/'/g, "&#39;");
  }

  function nmGetCurrentUser() {
    try {
      if (typeof currentUser !== "undefined" && currentUser) return currentUser;
    } catch (e) {}
    try {
      return JSON.parse(localStorage.getItem("nesru_current_user") || "null");
    } catch (e) {
      return null;
    }
  }

  // 1. Card Sign in / Register erga seenee booda Profile irratti akka dhokatu
  function nmSyncAuthBox() {
    const user = nmGetCurrentUser();
    let authCard = document.getElementById("auth-box-card");

    if (!authCard) {
      const authView = document.getElementById("view-auth");
      if (authView) {
        const forms = authView.querySelectorAll("form");
        for (const f of forms) {
          if (f.querySelector("#auth-email") && f.querySelector("#auth-password")) {
            authCard = f.closest(".card");
            if (authCard) authCard.id = "auth-box-card";
            break;
          }
        }
      }
    }

    if (authCard) {
      authCard.classList.toggle("hidden", Boolean(user));
    }

    const tag = document.getElementById("feed-user-tag");
    if (tag && user && user.name) {
      tag.textContent = user.name;
    }
  }

  // 2. Topbar irratti mallattoo Notification (🔔) dabaluu
  function nmEnsureBellButton() {
    const topbar = document.querySelector(".topbar");
    if (!topbar || document.getElementById("nm-bell-btn")) return;

    const btn = document.createElement("button");
    btn.type = "button";
    btn.id = "nm-bell-btn";
    btn.className = "btn btn-sm btn-outline";
    btn.style.position = "relative";
    btn.innerHTML = '🔔 <span id="nm-bell-badge" class="nav-badge hidden">0</span>';
    btn.onclick = nmOpenNotifModal;
    topbar.appendChild(btn);
    nmUpdateBellBadge();
  }

  function nmUpdateBellBadge() {
    const badge = document.getElementById("nm-bell-badge");
    if (!badge) return;
    const count = nmNotifs.filter((n) => !n.read).length;
    badge.textContent = String(count);
    badge.classList.toggle("hidden", count === 0);
  }

  function nmPushNotif(title, body, viewId) {
    nmNotifs.unshift({
      id: String(Date.now()) + Math.random().toString(36).slice(2, 6),
      title: title,
      body: body,
      viewId: viewId || "view-feed",
      read: false,
      time: new Date().toLocaleTimeString([], { hour: "2-digit", minute: "2-digit" })
    });
    if (nmNotifs.length > 40) nmNotifs.pop();
    localStorage.setItem(NOTIF_KEY, JSON.stringify(nmNotifs));
    nmUpdateBellBadge();

    if (typeof showToast === "function") {
      showToast("🔔 " + title + ": " + body);
    }
    if ("Notification" in window && Notification.permission === "granted") {
      try {
        new Notification(title, { body: body });
      } catch (e) {}
    }
  }

  window.nmOpenNotifModal = function () {
    if ("Notification" in window && Notification.permission === "default") {
      Notification.requestPermission().catch(function () {});
    }
    const modal = document.getElementById("nm-append-notif-modal");
    const list = document.getElementById("nm-append-notif-list");
    if (!modal || !list) return;

    if (!nmNotifs.length) {
      list.innerHTML = '<p class="muted">Beeksisni haaraan ammaaf hin jiru.</p>';
    } else {
      list.innerHTML = nmNotifs
        .map(
          (n) =>
            '<div style="padding:10px;border-bottom:1px solid #e2e8f0;border-radius:8px;margin-bottom:6px;cursor:pointer;background:' +
            (n.read ? "transparent" : "#eff6ff") +
            '" onclick="nmHandleNotifItem(\'' +
            nmEsc(n.id) +
            "','" +
            nmEsc(n.viewId) +
            "')\">" +
            '<div style="display:flex;justify-content:space-between;gap:8px;"><strong>' +
            nmEsc(n.title) +
            '</strong><span class="muted">' +
            nmEsc(n.time) +
            "</span></div>" +
            '<div style="margin-top:4px;">' +
            nmEsc(n.body) +
            "</div></div>"
        )
        .join("");
    }
    modal.classList.remove("hidden");
  };

  window.nmHandleNotifItem = function (id, viewId) {
    const found = nmNotifs.find((x) => x.id === id);
    if (found) found.read = true;
    localStorage.setItem(NOTIF_KEY, JSON.stringify(nmNotifs));
    nmUpdateBellBadge();
    nmCloseNotifModal();
    if (typeof navigateTo === "function" && viewId) {
      navigateTo(viewId);
    }
  };

  window.nmMarkAllRead = function () {
    nmNotifs.forEach((x) => {
      x.read = true;
    });
    localStorage.setItem(NOTIF_KEY, JSON.stringify(nmNotifs));
    nmUpdateBellBadge();
    nmOpenNotifModal();
  };

  window.nmCloseNotifModal = function () {
    const modal = document.getElementById("nm-append-notif-modal");
    if (modal) modal.classList.add("hidden");
  };

  // 3. Feed keessatti File Image Upload + Namoota Biroo (Users) dabaluu
  function nmEnsureFeedExtras() {
    const postForm = document.getElementById("post-form");
    const urlInput = document.getElementById("post-image");

    if (postForm && urlInput && !document.getElementById("nm-post-file-input")) {
      const wrap = document.createElement("div");
      wrap.className = "form-group";
      wrap.innerHTML =
        '<label for="nm-post-file-input">📷 Suuraa Filadhu / Upload Godhi</label>' +
        '<input type="file" id="nm-post-file-input" accept="image/*" />';
      urlInput.parentElement.parentNode.insertBefore(wrap, urlInput.parentElement);

      const fileInput = document.getElementById("nm-post-file-input");
      fileInput.addEventListener("change", function () {
        nmSelectedFile = (fileInput.files && fileInput.files[0]) || null;
        if (!nmSelectedFile) return;
        const reader = new FileReader();
        reader.onload = function (ev) {
          if (typeof ev.target.result === "string" && ev.target.result.length < 1800000) {
            urlInput.value = ev.target.result;
          }
        };
        reader.readAsDataURL(nmSelectedFile);
      });

      postForm.addEventListener("submit", function () {
        nmSelectedFile = null;
      });
    }

    const feedView = document.getElementById("view-feed");
    const postsList = document.getElementById("posts-list");
    if (feedView && postsList && !document.getElementById("nm-feed-users-card")) {
      const card = document.createElement("div");
      card.id = "nm-feed-users-card";
      card.className = "card";
      card.innerHTML =
        '<div style="display:flex;justify-content:space-between;align-items:center;margin-bottom:8px;">' +
        "<strong>👥 Namoota Biroo (Users)</strong>" +
        '<button type="button" class="btn btn-outline btn-sm" onclick="if(typeof navigateTo===\'function\')navigateTo(\'view-people\')">Hundaa →</button>' +
        '</div><div id="nm-feed-users-list"></div>';
      feedView.insertBefore(card, postsList);
    }

    nmRenderFeedUsers();
  }

  function nmRenderFeedUsers() {
    const listBox = document.getElementById("nm-feed-users-list");
    if (!listBox) return;

    const me = nmGetCurrentUser();
    const map = new Map();

    try {
      if (typeof state !== "undefined" && Array.isArray(state.profiles)) {
        state.profiles.forEach((p) => {
          if (p && p.email) map.set(p.email, p.full_name || p.email);
        });
      }
      if (typeof state !== "undefined" && Array.isArray(state.posts)) {
        state.posts.forEach((p) => {
          if (p && p.author_email && !map.has(p.author_email)) {
            map.set(p.author_email, p.author || p.author_email);
          }
        });
      }
    } catch (e) {}

    const others = Array.from(map.entries())
      .filter(([email]) => !me || email !== me.email)
      .slice(0, 6);

    if (!others.length) {
      listBox.innerHTML = '<p class="muted">Namoonni biroo yeroo galmaa\'an asitti mul\'atu.</p>';
      return;
    }

    listBox.innerHTML = others
      .map(
        ([email, name]) =>
          '<div class="person-card" style="margin-bottom:8px;padding:10px;">' +
          '<div><strong style="cursor:pointer;" onclick="if(typeof openPublicProfile===\'function\')openPublicProfile(\'' +
          nmEsc(email) +
          "')\">" +
          nmEsc(name) +
          '</strong><div class="muted">' +
          nmEsc(email) +
          "</div></div>" +
          '<div class="person-actions">' +
          '<button type="button" class="btn btn-sm btn-outline" onclick="if(typeof toggleFollow===\'function\')toggleFollow(\'' +
          nmEsc(email) +
          "')\">+ Follow</button>" +
          '<button type="button" class="btn btn-sm btn-outline" onclick="if(typeof openDirectChat===\'function\')openDirectChat(\'' +
          nmEsc(email) +
          "')\">💬</button>" +
          "</div></div>"
      )
      .join("");
  }

  // 4. Footer Copyright & Social Links yoo hin jiraanne dabaluu
  function nmEnsureFooter() {
    if (document.getElementById("nm-appended-footer")) return;
    const main = document.querySelector("main.container") || document.body;
    const footer = document.createElement("footer");
    footer.id = "nm-appended-footer";
    footer.className = "site-footer";
    footer.innerHTML =
      '<div class="social-row">' +
      '<a class="social-pill" href="https://nesru.dev" target="_blank" rel="noopener">🌐 Website</a>' +
      '<a class="social-pill" href="https://t.me/" target="_blank" rel="noopener">✈️ Telegram</a>' +
      '<a class="social-pill" href="https://facebook.com/" target="_blank" rel="noopener">📘 Facebook</a>' +
      '<a class="social-pill" href="https://youtube.com/" target="_blank" rel="noopener">▶️ YouTube</a>' +
      '<a class="social-pill" href="https://github.com/rammis-nas" target="_blank" rel="noopener">💻 GitHub</a>' +
      '<a class="social-pill" href="mailto:nesrumichael@gmail.com">✉️ Email</a>' +
      "</div>" +
      '<div class="copyright-line"><strong>&copy; 2026 Nesru Michael.</strong> Mirgi hundinuu seeraan eegamaadha.</div>';
    main.appendChild(footer);
  }

  window.addEventListener("DOMContentLoaded", function () {
    nmEnsureBellButton();
    nmEnsureFeedExtras();
    nmEnsureFooter();
    nmSyncAuthBox();
    setTimeout(function () {
      nmSyncAuthBox();
      nmRenderFeedUsers();
    }, 350);
  });

  document.addEventListener("click", function () {
    setTimeout(function () {
      nmSyncAuthBox();
      nmRenderFeedUsers();
    }, 180);
  });

  document.addEventListener("submit", function () {
    setTimeout(function () {
      nmSyncAuthBox();
      nmRenderFeedUsers();
      const me = nmGetCurrentUser();
      if (me) {
        nmPushNotif("Hojii Haaraa", "Jijjiiramni keessan galmaa'eera.", "view-feed");
      }
    }, 280);
  });
})();
</script>
Welcome to GitHub Docs! GitHub’s documentation is open source, meaning anyone from inside or outside the company can contribute. For full contributing guidelines, visit our [contributing guide](https://docs.github.com/en/contributing).


## Quick links by contributor type

* **Hubbers (GitHub employees):** See [CONTRIBUTING.md](https://github.com/github/docs-content/blob/main/CONTRIBUTING.md) in the `docs-content` repository for GitHub-specific processes.

* **Open source contributors:** See [CONTRIBUTING.md](https://github.com/github/docs/blob/main/.github/CONTRIBUTING.md) in the `docs` repository for a quick-start summary.

## How we sync changes across Docs repositories

There are two GitHub Docs repositories: 

- **`github/docs`** (public): Open to external contributions

- **`github/docs-internal`** (private): For GitHub employee contributions. 

The two repositories sync frequently. Content changes in one are reflected in the other.  Hubbers might prefer to post in `docs` when working with a customer, but `docs` has limitations on the types of contributions it accepts to safeguard the site and our workflows. Internal contributions should usually go to `docs-internal`.

**Important:** The `docs` repository accepts contributions to content files (`.md` files in `/content` and select `/data` sections like reusables only). Infrastructure files, workflows, and site-building code are not open for external modification.

## New to contributing

Here are some resources to help you get started with open source contributions:

* [Finding ways to contribute to open source on GitHub](https://docs.github.com/en/get-started/exploring-projects-on-github/finding-ways-to-contribute-to-open-source-on-github)
* [Set up Git](https://docs.github.com/en/get-started/git-basics/set-up-git)
* [GitHub flow](https://docs.github.com/en/get-started/using-github/github-flow)
* [How we use pull requests on Docs](https://docs.github.com/en/contributing/collaborating-on-github-docs/about-contributing-to-github-docs#pull-requests)

## License

This project is dual-licensed under:

* **Creative Commons Attribution 4.0** - for documentation and content in the assets, content, and data folders (see [LICENSE](LICENSE))
* **MIT License** - for code (see [LICENSE-CODE](LICENSE-CODE))
