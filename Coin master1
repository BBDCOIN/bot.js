/**
 * CoinMaster1 Telegram Mining Bot
 * Stack: Node.js + Telegraf + better-sqlite3
 * 
 * Setup:
 *   npm install telegraf better-sqlite3
 *   BOT_TOKEN=your_token node bot.js
 */

const { Telegraf, Markup } = require('telegraf');
const Database = require('better-sqlite3');

// ── Config ────────────────────────────────────────────────────────────────────
const BOT_TOKEN = 7797660415:AAETfQG3WESTwny1Xr1lfHSBOXNM-0EyUXQ || 'YOUR_BOT_TOKEN_HERE';
const MINI_APP_URL = process.env.MINI_APP_URL || 'https://your-domain.com'; // host your index.html here
const MINE_RATE = 10;          // coins per hour
const SPIN_COST = 50;          // coins per spin
const REFERRAL_BONUS = 100;    // coins for referrer
const CLAIM_INTERVAL_MS = 30 * 30 * 60; // 1 hour
const MIN_WITHDRAW = 2000;      // minimum coins to withdraw
const COINS_PER_USDT = 1000;   // exchange rate: 1000 coins = 1 USDT
const ADMIN_IDS = [123456789]; // ← replace with your Telegram user ID(s)

// ── Database ──────────────────────────────────────────────────────────────────
const db = new Database('coinmaster.db');

db.exec(`
  CREATE TABLE IF NOT EXISTS users (
    id          INTEGER PRIMARY KEY,
    username    TEXT,
    first_name  TEXT,
    coins       INTEGER DEFAULT 0,
    last_mine   INTEGER DEFAULT 0,
    spins       INTEGER DEFAULT 3,
    referred_by INTEGER DEFAULT NULL,
    referrals   INTEGER DEFAULT 0,
    joined_at   INTEGER DEFAULT (strftime('%s','now'))
  );

  CREATE TABLE IF NOT EXISTS withdrawals (
    id           INTEGER PRIMARY KEY AUTOINCREMENT,
    user_id      INTEGER NOT NULL,
    coins        INTEGER NOT NULL,
    usdt         REAL NOT NULL,
    wallet       TEXT NOT NULL,
    network      TEXT NOT NULL,
    status       TEXT DEFAULT 'pending',
    created_at   INTEGER DEFAULT (strftime('%s','now')),
    processed_at INTEGER DEFAULT NULL,
    admin_note   TEXT DEFAULT NULL
  );
`);

// ── DB helpers ────────────────────────────────────────────────────────────────
const getUser = (id) => db.prepare('SELECT * FROM users WHERE id = ?').get(id);

const upsertUser = (ctx) => {
  const { id, username, first_name } = ctx.from;
  const existing = getUser(id);
  if (!existing) {
    db.prepare(`INSERT OR IGNORE INTO users (id, username, first_name, last_mine)
                VALUES (?, ?, ?, ?)`)
      .run(id, username || '', first_name || 'Miner', Date.now());
  }
  return getUser(id);
};

const addCoins = (id, amount) =>
  db.prepare('UPDATE users SET coins = coins + ? WHERE id = ?').run(amount, id);

const setLastMine = (id, ts) =>
  db.prepare('UPDATE users SET last_mine = ? WHERE id = ?').run(ts, id);

const getLeaderboard = () =>
  db.prepare('SELECT first_name, username, coins, referrals FROM users ORDER BY coins DESC LIMIT 10').all();

// Withdrawal helpers
const createWithdrawal = (userId, coins, wallet, network) => {
  const usdt = coins / COINS_PER_USDT;
  db.prepare('UPDATE users SET coins = coins - ? WHERE id = ?').run(coins, userId);
  return db.prepare(
    'INSERT INTO withdrawals (user_id, coins, usdt, wallet, network) VALUES (?,?,?,?,?)'
  ).run(userId, coins, usdt, wallet, network);
};

const getUserWithdrawals = (userId) =>
  db.prepare('SELECT * FROM withdrawals WHERE user_id = ? ORDER BY created_at DESC LIMIT 5').all(userId);

const getPendingWithdrawals = () =>
  db.prepare(`SELECT w.*, u.first_name, u.username FROM withdrawals w
              JOIN users u ON w.user_id = u.id
              WHERE w.status = 'pending' ORDER BY w.created_at ASC`).all();

const updateWithdrawal = (id, status, note) =>
  db.prepare('UPDATE withdrawals SET status=?, admin_note=?, processed_at=? WHERE id=?')
    .run(status, note || null, Date.now(), id);

// Track multi-step withdraw conversation state
const withdrawState = {}; // { userId: { step, coins, wallet, network } }

// ── Mining logic ──────────────────────────────────────────────────────────────
const pendingCoins = (user) => {
  const elapsed = Date.now() - user.last_mine;
  const hours = elapsed / (1000 * 60 * 60);
  return Math.floor(hours * MINE_RATE);
};

// ── Spin logic ────────────────────────────────────────────────────────────────
const SPIN_RESULTS = [
  { emoji: '💰', label: 'Jackpot!', coins: 500, weight: 2 },
  { emoji: '⭐', label: 'Big Win!', coins: 200, weight: 5 },
  { emoji: '🪙', label: 'Win!',     coins: 100, weight: 15 },
  { emoji: '🎯', label: 'Nice!',    coins: 50,  weight: 25 },
  { emoji: '💎', label: 'Small Win',coins: 20,  weight: 30 },
  { emoji: '❌', label: 'Miss',     coins: 0,   weight: 23 },
];

const spinWheel = () => {
  const total = SPIN_RESULTS.reduce((s, r) => s + r.weight, 0);
  let rand = Math.random() * total;
  for (const result of SPIN_RESULTS) {
    rand -= result.weight;
    if (rand <= 0) return result;
  }
  return SPIN_RESULTS[SPIN_RESULTS.length - 1];
};

// ── Bot ───────────────────────────────────────────────────────────────────────
const bot = new Telegraf(BOT_TOKEN);

// /start
bot.start(async (ctx) => {
  const referrerId = ctx.startPayload ? parseInt(ctx.startPayload) : null;
  const user = upsertUser(ctx);

  // Process referral
  if (referrerId && referrerId !== ctx.from.id && !user.referred_by) {
    const referrer = getUser(referrerId);
    if (referrer) {
      db.prepare('UPDATE users SET referred_by = ? WHERE id = ?').run(referrerId, ctx.from.id);
      db.prepare('UPDATE users SET referrals = referrals + 1 WHERE id = ?').run(referrerId);
      addCoins(referrerId, REFERRAL_BONUS);
      addCoins(ctx.from.id, 50); // welcome bonus for new user
      try {
        await ctx.telegram.sendMessage(referrerId,
          `🎉 *${ctx.from.first_name}* joined using your referral link!\n+${REFERRAL_BONUS} coins added to your balance!`,
          { parse_mode: 'Markdown' });
      } catch (_) {}
    }
  }

  await ctx.replyWithMarkdown(
    `⛏️ *Welcome to CoinMaster1!*\n\n` +
    `Hello, *${ctx.from.first_name}*! Start mining coins, spin the wheel, and climb the leaderboard!\n\n` +
    `⚡ Mining rate: *${MINE_RATE} coins/hour*\n` +
    `🎰 Spin cost: *${SPIN_COST} coins*\n` +
    `👥 Referral bonus: *${REFERRAL_BONUS} coins*`,
    Markup.keyboard([
      ['⛏️ Mine', '🎰 Spin'],
      ['💰 Balance', '🏆 Leaderboard'],
      ['👥 Referral', '📊 Stats'],
      ['💸 Withdraw'],
    ]).resize()
  );

  // Show mini-app button
  await ctx.reply('Open the full game UI:', Markup.inlineKeyboard([
    Markup.button.webApp('🎮 Open CoinMaster1', MINI_APP_URL)
  ]));
});

// Mine
bot.hears('⛏️ Mine', async (ctx) => {
  const user = upsertUser(ctx);
  const earned = pendingCoins(user);

  if (earned === 0) {
    const waitMs = CLAIM_INTERVAL_MS - (Date.now() - user.last_mine);
    const waitMin = Math.ceil(waitMs / 60000);
    return ctx.reply(`⏳ Come back in *${waitMin} min* to collect more coins!`, { parse_mode: 'Markdown' });
  }

  addCoins(user.id, earned);
  setLastMine(user.id, Date.now());
  const updated = getUser(user.id);

  await ctx.replyWithMarkdown(
    `⛏️ *Mining Complete!*\n\n` +
    `+${earned} coins collected!\n` +
    `💰 Total balance: *${updated.coins} coins*`
  );
});

// Balance
bot.hears('💰 Balance', async (ctx) => {
  const user = upsertUser(ctx);
  const pending = pendingCoins(user);
  await ctx.replyWithMarkdown(
    `💰 *Your Balance*\n\n` +
    `Coins: *${user.coins}*\n` +
    `⏳ Pending: *${pending} coins*\n` +
    `🎰 Free spins: *${user.spins}*\n` +
    `👥 Referrals: *${user.referrals}*`
  );
});

// Spin
bot.hears('🎰 Spin', async (ctx) => {
  const user = upsertUser(ctx);

  if (user.spins <= 0 && user.coins < SPIN_COST) {
    return ctx.reply(`❌ Not enough coins! You need *${SPIN_COST} coins* to spin.\nMine more coins first! ⛏️`, { parse_mode: 'Markdown' });
  }

  // Deduct cost or free spin
  if (user.spins > 0) {
    db.prepare('UPDATE users SET spins = spins - 1 WHERE id = ?').run(user.id);
  } else {
    db.prepare('UPDATE users SET coins = coins - ? WHERE id = ?').run(SPIN_COST, user.id);
  }

  // Animate spin
  const spinMsg = await ctx.reply('🎰 Spinning...\n🔄 🔄 🔄');
  await new Promise(r => setTimeout(r, 1500));

  const result = spinWheel();
  if (result.coins > 0) addCoins(user.id, result.coins);

  const updated = getUser(user.id);

  await ctx.telegram.editMessageText(
    ctx.chat.id, spinMsg.message_id, undefined,
    `🎰 *Spin Result*\n\n${result.emoji} ${result.label}!\n` +
    (result.coins > 0 ? `+*${result.coins} coins!*\n` : `Better luck next time!\n`) +
    `\n💰 Balance: *${updated.coins} coins*`,
    { parse_mode: 'Markdown' }
  );
});

// Leaderboard
bot.hears('🏆 Leaderboard', async (ctx) => {
  const rows = getLeaderboard();
  const medals = ['🥇', '🥈', '🥉'];
  const lines = rows.map((r, i) => {
    const medal = medals[i] || `${i + 1}.`;
    const name = r.username ? `@${r.username}` : r.first_name;
    return `${medal} ${name} — *${r.coins}* coins`;
  });
  await ctx.replyWithMarkdown(`🏆 *Top Miners*\n\n${lines.join('\n')}`);
});

// Referral
bot.hears('👥 Referral', async (ctx) => {
  const user = upsertUser(ctx);
  const link = `https://t.me/${ctx.botInfo.username}?start=${user.id}`;
  await ctx.replyWithMarkdown(
    `👥 *Referral Program*\n\n` +
    `Invite friends and earn *${REFERRAL_BONUS} coins* per referral!\n\n` +
    `Your link:\n\`${link}\`\n\n` +
    `Total referrals: *${user.referrals}*`
  );
});

// Stats
bot.hears('📊 Stats', async (ctx) => {
  const user = upsertUser(ctx);
  const total = db.prepare('SELECT COUNT(*) as c FROM users').get().c;
  const rank = db.prepare('SELECT COUNT(*) as c FROM users WHERE coins > ?').get(user.coins).c + 1;
  await ctx.replyWithMarkdown(
    `📊 *Your Stats*\n\n` +
    `🏅 Rank: *#${rank}* of ${total} miners\n` +
    `💰 Coins: *${user.coins}*\n` +
    `👥 Referrals: *${user.referrals}*\n` +
    `🎰 Free spins: *${user.spins}*`
  );
});

// ── Withdrawal Flow ───────────────────────────────────────────────────────────

// Step 1: Show withdraw menu
bot.hears('💸 Withdraw', async (ctx) => {
  const user = upsertUser(ctx);
  const minUsdt = (MIN_WITHDRAW / COINS_PER_USDT).toFixed(2);
  const maxUsdt = (user.coins / COINS_PER_USDT).toFixed(2);
  const history = getUserWithdrawals(user.id);

  if (user.coins < MIN_WITHDRAW) {
    return ctx.replyWithMarkdown(
      `💸 *Withdrawal*\n\n` +
      `❌ Insufficient balance!\n\n` +
      `Minimum withdrawal: *${MIN_WITHDRAW} coins* (${minUsdt} USDT)\n` +
      `Your balance: *${user.coins} coins*\n\n` +
      `Keep mining to reach the minimum! ⛏️`
    );
  }

  let historyText = '';
  if (history.length > 0) {
    const statusEmoji = { pending: '⏳', approved: '✅', rejected: '❌' };
    historyText = '\n\n*Recent Withdrawals:*\n' + history.map(w =>
      `${statusEmoji[w.status] || '⏳'} ${w.coins} coins → ${w.usdt.toFixed(2)} USDT (${w.status})`
    ).join('\n');
  }

  withdrawState[user.id] = { step: 'amount' };

  await ctx.replyWithMarkdown(
    `💸 *Withdrawal*\n\n` +
    `💰 Balance: *${user.coins} coins* (≈ ${maxUsdt} USDT)\n` +
    `📊 Rate: *${COINS_PER_USDT} coins = 1 USDT*\n` +
    `⚠️ Minimum: *${MIN_WITHDRAW} coins*` +
    historyText +
    `\n\nHow many coins do you want to withdraw?\nType a number (min ${MIN_WITHDRAW}):`,
    Markup.keyboard([['❌ Cancel']]).resize()
  );
});

// Cancel
bot.hears('❌ Cancel', async (ctx) => {
  const userId = ctx.from.id;
  if (withdrawState[userId]) {
    delete withdrawState[userId];
    await ctx.reply('Withdrawal cancelled.', Markup.keyboard([
      ['⛏️ Mine', '🎰 Spin'],
      ['💰 Balance', '🏆 Leaderboard'],
      ['👥 Referral', '📊 Stats'],
      ['💸 Withdraw'],
    ]).resize());
  }
});

// Network selection callback
bot.action(/network:(.+)/, async (ctx) => {
  const userId = ctx.from.id;
  const ws = withdrawState[userId];
  if (!ws || ws.step !== 'network') return ctx.answerCbQuery('Session expired.');

  ws.network = ctx.match[1];
  ws.step = 'wallet';
  await ctx.answerCbQuery();
  await ctx.reply(`Enter your *${ws.network}* wallet address:`, { parse_mode: 'Markdown' });
});

// Confirm callback
bot.action('withdraw:confirm', async (ctx) => {
  const userId = ctx.from.id;
  const ws = withdrawState[userId];
  if (!ws || ws.step !== 'confirm') return ctx.answerCbQuery('Session expired.');

  const user = getUser(userId);
  if (user.coins < ws.coins) {
    delete withdrawState[userId];
    await ctx.answerCbQuery('Insufficient balance!');
    return ctx.reply('❌ Insufficient coins. Withdrawal cancelled.');
  }

  const result = createWithdrawal(userId, ws.coins, ws.wallet, ws.network);
  const wdId = result.lastInsertRowid;
  delete withdrawState[userId];

  await ctx.answerCbQuery('✅ Request submitted!');
  await ctx.reply(
    `✅ *Withdrawal Request #${wdId} Submitted!*\n\n` +
    `🪙 Coins deducted: *${ws.coins}*\n` +
    `💵 Amount: *${(ws.coins / COINS_PER_USDT).toFixed(2)} USDT*\n` +
    `🌐 Network: *${ws.network}*\n` +
    `👛 Wallet: \`${ws.wallet}\`\n\n` +
    `⏳ Processing time: 24-48 hours\nYou will be notified once processed.`,
    {
      parse_mode: 'Markdown',
      reply_markup: Markup.keyboard([
        ['⛏️ Mine', '🎰 Spin'],
        ['💰 Balance', '🏆 Leaderboard'],
        ['👥 Referral', '📊 Stats'],
        ['💸 Withdraw'],
      ]).resize().reply_markup
    }
  );

  // Notify admins
  const uname = ctx.from.username ? `@${ctx.from.username}` : ctx.from.first_name;
  for (const adminId of ADMIN_IDS) {
    try {
      await ctx.telegram.sendMessage(adminId,
        `🔔 *New Withdrawal Request #${wdId}*\n\n` +
        `👤 User: ${uname} (ID: ${userId})\n` +
        `🪙 Coins: ${ws.coins}\n` +
        `💵 USDT: ${(ws.coins / COINS_PER_USDT).toFixed(2)}\n` +
        `🌐 Network: ${ws.network}\n` +
        `👛 Wallet: \`${ws.wallet}\``,
        {
          parse_mode: 'Markdown',
          ...Markup.inlineKeyboard([
            [
              Markup.button.callback('✅ Approve', `admin:approve:${wdId}:${userId}`),
              Markup.button.callback('❌ Reject', `admin:reject:${wdId}:${userId}`)
            ]
          ])
        }
      );
    } catch (_) {}
  }
});

bot.action('withdraw:cancel_confirm', async (ctx) => {
  const userId = ctx.from.id;
  delete withdrawState[userId];
  await ctx.answerCbQuery('Cancelled.');
  await ctx.reply('Withdrawal cancelled.', Markup.keyboard([
    ['⛏️ Mine', '🎰 Spin'],
    ['💰 Balance', '🏆 Leaderboard'],
    ['👥 Referral', '📊 Stats'],
    ['💸 Withdraw'],
  ]).resize());
});

// ── Admin: Approve / Reject ───────────────────────────────────────────────────
bot.action(/admin:(approve|reject):(\d+):(\d+)/, async (ctx) => {
  if (!ADMIN_IDS.includes(ctx.from.id)) return ctx.answerCbQuery('Not authorized.');
  const action = ctx.match[1];
  const wdId = parseInt(ctx.match[2]);
  const userId = parseInt(ctx.match[3]);

  const wd = db.prepare('SELECT * FROM withdrawals WHERE id = ?').get(wdId);
  if (!wd) return ctx.answerCbQuery('Withdrawal not found.');
  if (wd.status !== 'pending') return ctx.answerCbQuery(`Already ${wd.status}.`);

  if (action === 'approve') {
    updateWithdrawal(wdId, 'approved', 'Approved by admin');
    await ctx.answerCbQuery('✅ Approved!');
    await ctx.editMessageText(ctx.callbackQuery.message.text + '\n\n✅ *APPROVED*', { parse_mode: 'Markdown' });
    try {
      await ctx.telegram.sendMessage(userId,
        `✅ *Withdrawal #${wdId} Approved!*\n\n` +
        `💵 *${(wd.usdt).toFixed(2)} USDT* has been sent to:\n\`${wd.wallet}\`\n\nThank you! 🎉`,
        { parse_mode: 'Markdown' });
    } catch (_) {}
  } else {
    updateWithdrawal(wdId, 'rejected', 'Rejected by admin');
    // Refund coins
    addCoins(userId, wd.coins);
    await ctx.answerCbQuery('❌ Rejected & refunded.');
    await ctx.editMessageText(ctx.callbackQuery.message.text + '\n\n❌ *REJECTED — coins refunded*', { parse_mode: 'Markdown' });
    try {
      await ctx.telegram.sendMessage(userId,
        `❌ *Withdrawal #${wdId} Rejected*\n\n` +
        `Your *${wd.coins} coins* have been refunded to your balance.\n` +
        `Please contact support if you have questions.`,
        { parse_mode: 'Markdown' });
    } catch (_) {}
  }
});

// ── Admin: List pending withdrawals ──────────────────────────────────────────
bot.command('pending', async (ctx) => {
  if (!ADMIN_IDS.includes(ctx.from.id)) return;
  const list = getPendingWithdrawals();
  if (!list.length) return ctx.reply('No pending withdrawals ✅');

  for (const w of list) {
    const uname = w.username ? `@${w.username}` : w.first_name;
    await ctx.replyWithMarkdown(
      `⏳ *Withdrawal #${w.id}*\n` +
      `👤 ${uname} (${w.user_id})\n` +
      `🪙 ${w.coins} coins → ${w.usdt.toFixed(2)} USDT\n` +
      `🌐 ${w.network} | \`${w.wallet}\``,
      Markup.inlineKeyboard([
        [
          Markup.button.callback('✅ Approve', `admin:approve:${w.id}:${w.user_id}`),
          Markup.button.callback('❌ Reject', `admin:reject:${w.id}:${w.user_id}`)
        ]
      ])
    );
  }
});

// ── Text message handler (for multi-step withdrawal) ─────────────────────────
bot.on('text', async (ctx) => {
  const userId = ctx.from.id;
  const ws = withdrawState[userId];
  if (!ws) return;

  const text = ctx.message.text.trim();

  if (ws.step === 'amount') {
    const coins = parseInt(text);
    const user = getUser(userId);
    if (isNaN(coins) || coins < MIN_WITHDRAW) {
      return ctx.reply(`❌ Enter a valid amount (minimum ${MIN_WITHDRAW} coins).`);
    }
    if (coins > user.coins) {
      return ctx.reply(`❌ You only have ${user.coins} coins.`);
    }
    ws.coins = coins;
    ws.step = 'network';
    await ctx.replyWithMarkdown(
      `💵 Amount: *${coins} coins* = *${(coins / COINS_PER_USDT).toFixed(2)} USDT*\n\nSelect network:`,
      Markup.inlineKeyboard([
        [Markup.button.callback('🔵 TRC20 (TRON)', 'network:TRC20')],
        [Markup.button.callback('🟣 BEP20 (BSC)', 'network:BEP20')],
        [Markup.button.callback('🔷 ERC20 (ETH)', 'network:ERC20')],
      ])
    );

  } else if (ws.step === 'wallet') {
    if (text.length < 20) {
      return ctx.reply('❌ Invalid wallet address. Please try again.');
    }
    ws.wallet = text;
    ws.step = 'confirm';
    await ctx.replyWithMarkdown(
      `📋 *Confirm Withdrawal*\n\n` +
      `🪙 Coins: *${ws.coins}*\n` +
      `💵 USDT: *${(ws.coins / COINS_PER_USDT).toFixed(2)}*\n` +
      `🌐 Network: *${ws.network}*\n` +
      `👛 Wallet: \`${ws.wallet}\`\n\n` +
      `⚠️ Please double-check your wallet address. Transactions cannot be reversed!`,
      Markup.inlineKeyboard([
        [
          Markup.button.callback('✅ Confirm', 'withdraw:confirm'),
          Markup.button.callback('❌ Cancel', 'withdraw:cancel_confirm')
        ]
      ])
    );
  }
});

// Free spins every 24h
setInterval(() => {
  db.prepare('UPDATE users SET spins = MIN(spins + 1, 5)').run();
}, 24 * 60 * 60 * 1000);

bot.launch().then(() => console.log('🤖 CoinMaster1 bot is running!'));
process.once('SIGINT', () => bot.stop('SIGINT'));
process.once('SIGTERM', () => bot.stop('SIGTERM'));
