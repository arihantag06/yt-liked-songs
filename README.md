# yt-liked-songs
chrome script to copy liked songs on yt music


```
(() => {
  const TARGET_PLAYLIST = "my liked songs"; // change to exact visible playlist name if needed
  const ROW_SELECTORS = [
    "ytmusic-responsive-list-item-renderer",
    "ytmusic-two-row-item-renderer",
    "ytmusic-list-item-renderer",
  ];

  const MENU_TEXTS = ["Add to playlist", "Save to playlist"];
  const WAIT_MS = 12000;

  const sleep = (ms) => new Promise((r) => setTimeout(r, ms));

  function isVisible(el) {
    return !!el && el.offsetParent !== null;
  }

  function waitFor(fn, timeout = WAIT_MS, step = 100) {
    return new Promise((resolve, reject) => {
      const start = Date.now();

      (function poll() {
        const value = fn();
        if (value) return resolve(value);
        if (Date.now() - start > timeout) return reject(new Error("Timeout"));
        setTimeout(poll, step);
      })();
    });
  }

  function getRows() {
    for (const sel of ROW_SELECTORS) {
      const rows = Array.from(document.querySelectorAll(sel));
      if (rows.length) return rows;
    }
    return [];
  }

  function getTitle(row) {
    const el =
      row.querySelector("a[title]") ||
      row.querySelector("#title") ||
      row.querySelector(".title") ||
      row.querySelector(".yt-simple-endpoint");
    return el ? el.textContent.trim() : "(unknown title)";
  }

  function getMenuButton(row) {
    const buttons = Array.from(
      row.querySelectorAll('button, yt-icon-button, tp-yt-paper-icon-button, [role="button"]')
    );

    for (const el of buttons) {
      const label = (
        el.getAttribute("aria-label") ||
        el.getAttribute("title") ||
        el.textContent ||
        ""
      ).toLowerCase();

      if (
        label.includes("more") ||
        label.includes("options") ||
        label.includes("menu") ||
        label.includes("overflow")
      ) {
        return el;
      }
    }

    return buttons[buttons.length - 1] || null;
  }

  function realisticClick(el) {
    if (!el) return false;
    el.scrollIntoView({ block: "center", inline: "center" });
    el.dispatchEvent(new MouseEvent("mouseover", { bubbles: true, cancelable: true, view: window }));
    el.dispatchEvent(new MouseEvent("mousedown", { bubbles: true, cancelable: true, view: window }));
    el.dispatchEvent(new MouseEvent("mouseup", { bubbles: true, cancelable: true, view: window }));
    el.dispatchEvent(new MouseEvent("click", { bubbles: true, cancelable: true, view: window }));
    return true;
  }

  function findVisibleText(text) {
    const candidates = Array.from(document.querySelectorAll("yt-formatted-string, span, tp-yt-paper-item, div"))
      .filter(isVisible);

    return candidates.find((el) => el.textContent.trim() === text) || null;
  }

  async function addRowToPlaylist(row, index, total) {
    const title = getTitle(row);
    const menuButton = getMenuButton(row);

    if (!menuButton) {
      console.warn("No menu button found for row", index, title);
      return;
    }

    console.log(`Processing ${index + 1}/${total}: ${title}`);
    realisticClick(menuButton);

    let addItem = null;
    try {
      addItem = await waitFor(() => {
        for (const t of MENU_TEXTS) {
          const el = findVisibleText(t);
          if (el) return el;
        }
        return null;
      });
    } catch (e) {
      console.warn("Menu did not open or menu item not found for:", title);
      return;
    }

    realisticClick(addItem);
    await sleep(400);

    let playlistItem = null;
    try {
      playlistItem = await waitFor(() => findVisibleText(TARGET_PLAYLIST));
    } catch (e) {
      console.warn("Target playlist not found:", TARGET_PLAYLIST);
      return;
    }

    realisticClick(playlistItem);
    await sleep(500);
    document.body.click();
    await sleep(200);
  }

  async function run() {
    const rows = getRows();
    if (!rows.length) {
      console.warn("No song rows found.");
      return;
    }

    console.log("Found rows:", rows.length);

    for (let i = 0; i < rows.length; i++) {
      await addRowToPlaylist(rows[i], i, rows.length);
      await sleep(800);
    }

    console.log("Done.");
  }

  run();
})();
```
