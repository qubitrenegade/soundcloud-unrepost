```javascript
/*
README: SoundCloud Un-Repost Script
====================================

What is this?
-------------
This script automates the process of removing reposts from your SoundCloud profile.
It will go through your reposts one by one, click the "Unpost" button, and
confirm the action, effectively cleaning up your feed.

How to Use
----------
1.  Go to your SoundCloud reposts page (e.g., https://soundcloud.com/your-username/reposts).
2.  Open the Developer Console in your browser.
    - Chrome: `Ctrl+Shift+J` (Windows/Linux) or `Cmd+Option+J` (Mac)
    - Firefox: `Ctrl+Shift+I` (Windows/Linux) or `Cmd+Option+I` (Mac), then select the "Console" tab.
3.  Scroll down a bit on the page to load the first few tracks.
4.  Copy the ENTIRE code block below (from `(async () => {` to the final `})();`).
5.  Paste the code into the console and press `Enter`.
6.  The script will now run in the console tab. You can watch its progress there.

Configuration
-------------
Before you run the script, you can change the settings below to customize it.

- `EXCEPTIONS`: This is the most important part! It's a "keep list". Any repost
  that matches a term in this list will be SKIPPED. It uses regular expressions,
  which are flexible for matching text.

  **To add an exception, add a new line inside the `[]` brackets.**

  - To skip any track with the word "ArtistName" in it (case-insensitive):
    /ArtistName/i,

  - To skip a specific track title like "My Favorite Song (Remix)":
    /My Favorite Song.*Remix/i,

- `LIMIT`: A safety feature. This is the maximum number of reposts the script
  will remove. It's set to 10 by default to prevent accidents.
  Set to `Infinity` to remove all reposts (that don't match your exceptions).

Disclaimer
----------
This is an unofficial script and is shared for educational purposes only. It relies
on SoundCloud's current website structure, which can change at any time. If
SoundCloud updates its site, this script may stop working.

I originally created and used this script to help with my own rebranding and found
it helpful, so I'm sharing it in the hope that it might be useful to others.
However, it comes with absolutely no warranty, expressed or implied. Use it
entirely at your own risk.

*Lasciate ogne speranza, voi ch'intrate.*

*/

(async () => {
  //================================================================================
  // SCRIPT CONFIGURATION - Change these values to your liking
  //================================================================================

  // --- Items to KEEP ---
  // Add any artists, titles, or keywords you want to AVOID deleting.
  const EXCEPTIONS = [
    /Phibes/i,                              // Example: Keeps any track with "Phibes"
    /Shoot\s*To\s*Thrill.*Avoision.*Bootleg/i // Example: Keeps a specific track
    // Add your own exceptions here, like this:
    // /MyFavoriteArtist/i,
  ];

  // --- Safety Limit ---
  // The script will stop after deleting this many reposts.
  // Set to Infinity to run until it's done: const LIMIT = Infinity;
  const LIMIT = 10;

  // --- Timings (in milliseconds) ---
  // You can increase these if the script feels too fast for your connection.
  const OPEN_MENU_DELAY = 200;    // Time to wait after clicking the '...' menu
  const CLICK_DELETE_DELAY = 250; // Time to wait after clicking the final 'Unpost' button
  const SCROLL_DELAY = 600;       // Time to wait for new items to load after scrolling

  //================================================================================
  // SCRIPT LOGIC - No need to edit below this line
  //================================================================================

  console.log(`🚀 Starting Un-Repost script...`);
  console.log(`- Limit: ${LIMIT} reposts`);
  console.log(`- Keeping exceptions:`, EXCEPTIONS.map(r => r.toString()));

  const SEL = {
    item: 'li.soundList__item, article.audible',
    menuBtn: 'button[aria-label*="Unpost"], button.sc-button-repost.sc-button-selected',
    deleteBtn: 'button.repostOverlay__formButtonDelete, button.sc-button-nostyle.repostOverlay__formButtonDelete'
  };

  const sleep = ms => new Promise(r => setTimeout(r, ms));
  const isVisible = el => el && el.offsetParent !== null;

  function shouldSkip(item) {
    const textContent = (item.textContent || '').replace(/\s+/g, ' ');
    const matchedException = EXCEPTIONS.find(rx => rx.test(textContent));
    if (matchedException) {
      console.log(`🚫 Skipping item because it matches exception: ${matchedException.toString()}`, item);
      return true;
    }
    return false;
  }

  async function waitForDeleteButton(timeout = 5000) {
    const startTime = Date.now();
    while (Date.now() - startTime < timeout) {
      const btn = document.querySelector(SEL.deleteBtn);
      if (isVisible(btn)) return btn;
      await sleep(50);
    }
    return null;
  }

  async function processOne() {
    const items = Array.from(document.querySelectorAll(SEL.item));
    const unprocessedItems = items.filter(it => !it.hasAttribute('data-processed'));

    if (unprocessedItems.length === 0) return 'NO_ITEMS';

    for (const item of unprocessedItems) {
      item.setAttribute('data-processed', 'true'); // Mark as processed to avoid loops
      const menuButton = item.querySelector(SEL.menuBtn);

      if (!menuButton) continue;
      if (shouldSkip(item)) continue;

      const trackInfo = item.querySelector('a.soundTitle__title > span')?.textContent || 'Unknown Track';
      console.log(`🗑️ Processing: "${trackInfo}"`);

      menuButton.click();
      await sleep(OPEN_MENU_DELAY);

      const deleteButton = await waitForDeleteButton();
      if (!deleteButton) {
        console.warn('Could not find the final "Unpost" button. Clicking menu button again to close.', item);
        // Attempt to close the menu to prevent getting stuck
        menuButton.click();
        return false;
      }

      deleteButton.click();
      console.log(`✅ Successfully un-reposted "${trackInfo}".`);
      await sleep(CLICK_DELETE_DELAY);
      return true;
    }
    return 'ALL_SKIPPED'; // Found items, but they were all exceptions
  }

  let deletedCount = 0;
  while (deletedCount < LIMIT) {
    const result = await processOne();

    if (result === true) {
      deletedCount++;
      continue;
    }

    if (result === 'ALL_SKIPPED') {
        // All visible items were skipped, we still need to scroll to find more
        console.log("All visible items were exceptions, scrolling to find more...");
    }

    // If no unprocessed items were found, try to load more
    const scrollPosBefore = window.scrollY;
    window.scrollTo({ top: document.body.scrollHeight, behavior: 'instant' });
    await sleep(SCROLL_DELAY);
    const scrollPosAfter = window.scrollY;

    // If scrolling didn't change the page position, we've hit the end.
    if (scrollPosBefore === scrollPosAfter) {
      console.log('🏁 Reached the end of the page.');
      break;
    }
  }

  console.log(`\n🎉 Done. Deleted ${deletedCount} repost(s).`);
})();
```
