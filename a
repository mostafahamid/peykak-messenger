// worker.js - نسخه کامل با نوتیفیکیشن، گروه عمومی، تیک آبی، آنلاین/آفلاین، آخرین بازدید تهران، تغییر نام نمایشی، پنل ادمین، نمایش پیام‌های خوانده نشده و حرکات دست

export default {
  async fetch(request, env, ctx) {
    await initDB(env);
    
    const url = new URL(request.url);
    
    if (url.pathname === "/") {
      return new Response(getHTML(), {
        headers: { "Content-Type": "text/html; charset=utf-8" }
      });
    }
    
    if (url.pathname === "/api/messages" && request.method === "GET") {
      try {
        const { searchParams } = new URL(request.url);
        const user1 = searchParams.get('user1');
        const user2 = searchParams.get('user2');
        const since = searchParams.get('since');
        
        let query = "SELECT * FROM messages WHERE is_deleted = 0 AND ";
        let params = [];
        
        if (user1 && user2) {
          if (since) {
            query += "((sender = ? AND receiver = ?) OR (sender = ? AND receiver = ?)) AND id > ? ORDER BY created_at ASC LIMIT 200";
            params = [user1, user2, user2, user1, since];
          } else {
            query += "(sender = ? AND receiver = ?) OR (sender = ? AND receiver = ?) ORDER BY created_at ASC LIMIT 200";
            params = [user1, user2, user2, user1];
          }
        } else {
          query += "1=1 ORDER BY created_at ASC LIMIT 200";
        }
        
        const result = await env.DB.prepare(query).bind(...params).all();
        
        return new Response(JSON.stringify(result.results), {
          headers: { "Content-Type": "application/json" }
        });
      } catch (error) {
        return new Response(JSON.stringify({ error: error.message }), { 
          status: 500,
          headers: { "Content-Type": "application/json" }
        });
      }
    }
    
    if (url.pathname === "/api/unread-counts" && request.method === "GET") {
      try {
        const receiver = url.searchParams.get('receiver');
        if (!receiver) return new Response(JSON.stringify({}), { headers: { "Content-Type": "application/json" } });
        const result = await env.DB.prepare(
          "SELECT sender, COUNT(*) as count FROM messages WHERE receiver = ? AND is_read = 0 AND is_deleted = 0 GROUP BY sender"
        ).bind(receiver).all();
        const counts = {};
        for (const row of result.results) counts[row.sender] = row.count;
        return new Response(JSON.stringify(counts), { headers: { "Content-Type": "application/json" } });
      } catch (error) {
        return new Response(JSON.stringify({}), { headers: { "Content-Type": "application/json" } });
      }
    }

    if (url.pathname === "/api/messages" && request.method === "POST") {
      try {
        const { text, sender, receiver, image_data, voice_data, reply_to_id } = await request.json();
        
        const result = await env.DB.prepare(
          "INSERT INTO messages (text, sender, receiver, image_data, voice_data, reply_to_id) VALUES (?, ?, ?, ?, ?, ?) RETURNING *"
        ).bind(text || "", sender, receiver, image_data || null, voice_data || null, reply_to_id || null).run();
        
        return new Response(JSON.stringify(result.results[0]), {
          headers: { "Content-Type": "application/json" }
        });
      } catch (error) {
        return new Response(JSON.stringify({ error: error.message }), { 
          status: 500,
          headers: { "Content-Type": "application/json" }
        });
      }
    }
    
    if (url.pathname === "/api/delete-message" && request.method === "POST") {
      try {
        const { messageId, sender } = await request.json();
        await env.DB.prepare(
          "UPDATE messages SET is_deleted = 1 WHERE id = ? AND sender = ?"
        ).bind(messageId, sender).run();
        return new Response(JSON.stringify({ success: true }));
      } catch (error) {
        return new Response(JSON.stringify({ error: error.message }), { status: 500 });
      }
    }
    
    if (url.pathname === "/api/edit-message" && request.method === "POST") {
      try {
        const { messageId, sender, newText } = await request.json();
        await env.DB.prepare(
          "UPDATE messages SET text = ?, is_edited = 1 WHERE id = ? AND sender = ?"
        ).bind(newText, messageId, sender).run();
        return new Response(JSON.stringify({ success: true }));
      } catch (error) {
        return new Response(JSON.stringify({ error: error.message }), { status: 500 });
      }
    }
    
    if (url.pathname === "/api/mark-read" && request.method === "POST") {
      try {
        const { sender, receiver } = await request.json();
        await env.DB.prepare("UPDATE messages SET is_read = 1 WHERE sender = ? AND receiver = ? AND is_read = 0").bind(sender, receiver).run();
        return new Response(JSON.stringify({ success: true }));
      } catch (error) {
        return new Response(JSON.stringify({ error: error.message }), { status: 500 });
      }
    }
    
    if (url.pathname === "/api/update-last-seen" && request.method === "POST") {
      try {
        const { username } = await request.json();
        await env.DB.prepare("UPDATE users SET last_seen = CURRENT_TIMESTAMP WHERE username = ?").bind(username).run();
        return new Response(JSON.stringify({ success: true }));
      } catch (error) {
        return new Response(JSON.stringify({ error: error.message }), { status: 500 });
      }
    }
    
    if (url.pathname === "/api/register" && request.method === "POST") {
      try {
        const { username, password, displayName } = await request.json();
        const existing = await env.DB.prepare("SELECT * FROM users WHERE username = ?").bind(username).first();
        if (existing) {
          return new Response(JSON.stringify({ success: false, error: "این نام کاربری قبلاً ثبت شده است" }));
        }
        await env.DB.prepare("INSERT INTO users (username, password, display_name) VALUES (?, ?, ?)").bind(username, password, displayName || username).run();
        return new Response(JSON.stringify({ success: true, user: { username: username, name: displayName || username, avatar: "👤" } }));
      } catch (error) {
        return new Response(JSON.stringify({ success: false, error: error.message }), { status: 500 });
      }
    }
    
    if (url.pathname === "/api/login" && request.method === "POST") {
      const { username, password } = await request.json();
      const user = await env.DB.prepare("SELECT * FROM users WHERE username = ? AND password = ?").bind(username, password).first();
      if (user) {
        return new Response(JSON.stringify({ success: true, user: { username: user.username, name: user.display_name, avatar: user.avatar || "👤" } }));
      }
      return new Response(JSON.stringify({ success: false, error: "نام کاربری یا رمز عبور اشتباه است" }));
    }
    
    if (url.pathname === "/api/change-password" && request.method === "POST") {
      try {
        const { username, oldPassword, newPassword } = await request.json();
        const user = await env.DB.prepare("SELECT * FROM users WHERE username = ? AND password = ?").bind(username, oldPassword).first();
        if (!user) {
          return new Response(JSON.stringify({ success: false, error: "رمز عبور فعلی اشتباه است" }));
        }
        await env.DB.prepare("UPDATE users SET password = ? WHERE username = ?").bind(newPassword, username).run();
        return new Response(JSON.stringify({ success: true, message: "رمز عبور با موفقیت تغییر کرد" }));
      } catch (error) {
        return new Response(JSON.stringify({ success: false, error: error.message }), { status: 500 });
      }
    }
    
    if (url.pathname === "/api/update-display-name" && request.method === "POST") {
      try {
        const { username, displayName } = await request.json();
        if (!displayName || displayName.trim() === "") {
          return new Response(JSON.stringify({ success: false, error: "نام نمایشی نمی‌تواند خالی باشد" }));
        }
        await env.DB.prepare("UPDATE users SET display_name = ? WHERE username = ?").bind(displayName, username).run();
        return new Response(JSON.stringify({ success: true, displayName: displayName }));
      } catch (error) {
        return new Response(JSON.stringify({ success: false, error: error.message }), { status: 500 });
      }
    }
    
    if (url.pathname === "/api/delete-account" && request.method === "POST") {
      try {
        const { username, password } = await request.json();
        const user = await env.DB.prepare("SELECT * FROM users WHERE username = ? AND password = ?").bind(username, password).first();
        if (!user) {
          return new Response(JSON.stringify({ success: false, error: "رمز عبور اشتباه است" }));
        }
        await env.DB.prepare("DELETE FROM messages WHERE sender = ? OR receiver = ?").bind(username, username).run();
        await env.DB.prepare("DELETE FROM group_messages WHERE sender = ?").bind(username).run();
        await env.DB.prepare("DELETE FROM users WHERE username = ?").bind(username).run();
        return new Response(JSON.stringify({ success: true, message: "حساب کاربری شما حذف شد" }));
      } catch (error) {
        return new Response(JSON.stringify({ success: false, error: error.message }), { status: 500 });
      }
    }

    if (url.pathname === "/api/admin-delete-user" && request.method === "POST") {
      try {
        const { adminUsername, targetUsername } = await request.json();
        if (adminUsername !== "mostafa") {
          return new Response(JSON.stringify({ success: false, error: "دسترسی غیرمجاز" }), { status: 403 });
        }
        if (targetUsername === "mostafa") {
          return new Response(JSON.stringify({ success: false, error: "نمی‌توانید حساب ادمین را حذف کنید" }));
        }
        await env.DB.prepare("DELETE FROM messages WHERE sender = ? OR receiver = ?").bind(targetUsername, targetUsername).run();
        await env.DB.prepare("DELETE FROM group_messages WHERE sender = ?").bind(targetUsername).run();
        await env.DB.prepare("DELETE FROM users WHERE username = ?").bind(targetUsername).run();
        return new Response(JSON.stringify({ success: true }));
      } catch (error) {
        return new Response(JSON.stringify({ success: false, error: error.message }), { status: 500 });
      }
    }
    
    if (url.pathname === "/api/users" && request.method === "GET") {
      try {
        const currentUser = url.searchParams.get('current');
        const result = await env.DB.prepare("SELECT username, display_name, last_seen, avatar, bio FROM users WHERE username != ?").bind(currentUser).all();
        return new Response(JSON.stringify(result.results), { headers: { "Content-Type": "application/json" } });
      } catch (error) {
        return new Response(JSON.stringify([]));
      }
    }

    if (url.pathname === "/api/update-avatar" && request.method === "POST") {
      try {
        const { username, avatar } = await request.json();
        await env.DB.prepare("UPDATE users SET avatar = ? WHERE username = ?").bind(avatar, username).run();
        return new Response(JSON.stringify({ success: true }), { headers: { "Content-Type": "application/json" } });
      } catch (error) {
        return new Response(JSON.stringify({ success: false, error: error.message }), { status: 500 });
      }
    }

    if (url.pathname === "/api/update-bio" && request.method === "POST") {
      try {
        const { username, bio } = await request.json();
        await env.DB.prepare("UPDATE users SET bio = ? WHERE username = ?").bind(bio || null, username).run();
        return new Response(JSON.stringify({ success: true }), { headers: { "Content-Type": "application/json" } });
      } catch (error) {
        return new Response(JSON.stringify({ success: false, error: error.message }), { status: 500 });
      }
    }

    if (url.pathname === "/api/get-avatar" && request.method === "GET") {
      try {
        const username = url.searchParams.get('username');
        const user = await env.DB.prepare("SELECT avatar, bio FROM users WHERE username = ?").bind(username).first();
        return new Response(JSON.stringify({ avatar: user ? user.avatar : null, bio: user ? user.bio : null }), { headers: { "Content-Type": "application/json" } });
      } catch (error) {
        return new Response(JSON.stringify({ avatar: null, bio: null }));
      }
    }

    if (url.pathname === "/api/group-messages" && request.method === "GET") {
      try {
        const since = url.searchParams.get('since');
        let query, params;
        if (since) {
          query = "SELECT * FROM group_messages WHERE is_deleted = 0 AND id > ? ORDER BY created_at ASC LIMIT 200";
          params = [since];
        } else {
          query = "SELECT * FROM group_messages WHERE is_deleted = 0 ORDER BY created_at ASC LIMIT 200";
          params = [];
        }
        const result = await env.DB.prepare(query).bind(...params).all();
        return new Response(JSON.stringify(result.results), { headers: { "Content-Type": "application/json" } });
      } catch (error) {
        return new Response(JSON.stringify({ error: error.message }), { status: 500, headers: { "Content-Type": "application/json" } });
      }
    }

    if (url.pathname === "/api/group-messages" && request.method === "POST") {
      try {
        const { text, sender, image_data, voice_data, reply_to_id } = await request.json();
        const result = await env.DB.prepare(
          "INSERT INTO group_messages (text, sender, image_data, voice_data, reply_to_id) VALUES (?, ?, ?, ?, ?) RETURNING *"
        ).bind(text || "", sender, image_data || null, voice_data || null, reply_to_id || null).run();
        return new Response(JSON.stringify(result.results[0]), { headers: { "Content-Type": "application/json" } });
      } catch (error) {
        return new Response(JSON.stringify({ error: error.message }), { status: 500, headers: { "Content-Type": "application/json" } });
      }
    }

    if (url.pathname === "/api/delete-group-message" && request.method === "POST") {
      try {
        const { messageId, sender } = await request.json();
        await env.DB.prepare("UPDATE group_messages SET is_deleted = 1 WHERE id = ? AND sender = ?").bind(messageId, sender).run();
        return new Response(JSON.stringify({ success: true }));
      } catch (error) {
        return new Response(JSON.stringify({ error: error.message }), { status: 500 });
      }
    }

    if (url.pathname === "/api/edit-group-message" && request.method === "POST") {
      try {
        const { messageId, sender, newText } = await request.json();
        await env.DB.prepare("UPDATE group_messages SET text = ?, is_edited = 1 WHERE id = ? AND sender = ?").bind(newText, messageId, sender).run();
        return new Response(JSON.stringify({ success: true }));
      } catch (error) {
        return new Response(JSON.stringify({ error: error.message }), { status: 500 });
      }
    }

    if (url.pathname === "/api/posts" && request.method === "GET") {
      try {
        const viewer = url.searchParams.get('viewer') || '';
        const since = url.searchParams.get('since');
        let query, params;
        if (since) {
          query = "SELECT * FROM posts WHERE is_deleted = 0 AND id > ? ORDER BY created_at DESC LIMIT 50";
          params = [since];
        } else {
          query = "SELECT * FROM posts WHERE is_deleted = 0 ORDER BY created_at DESC LIMIT 50";
          params = [];
        }
        const result = await env.DB.prepare(query).bind(...params).all();
        const posts = result.results;
        if (viewer && posts.length) {
          const likeChecks = await Promise.all(posts.map(p =>
            env.DB.prepare("SELECT 1 FROM post_likes WHERE post_id = ? AND username = ?").bind(p.id, viewer).first()
          ));
          posts.forEach((p, i) => { p.liked_by_me = !!likeChecks[i]; });
        }
        return new Response(JSON.stringify(posts), { headers: { "Content-Type": "application/json" } });
      } catch (error) {
        return new Response(JSON.stringify({ error: error.message }), { status: 500, headers: { "Content-Type": "application/json" } });
      }
    }

    if (url.pathname === "/api/posts" && request.method === "POST") {
      try {
        const { text, sender, image_data } = await request.json();
        if (!text && !image_data) return new Response(JSON.stringify({ error: "محتوا الزامی است" }), { status: 400 });
        const result = await env.DB.prepare(
          "INSERT INTO posts (text, sender, image_data) VALUES (?, ?, ?) RETURNING *"
        ).bind(text || "", sender, image_data || null).run();
        return new Response(JSON.stringify(result.results[0]), { headers: { "Content-Type": "application/json" } });
      } catch (error) {
        return new Response(JSON.stringify({ error: error.message }), { status: 500, headers: { "Content-Type": "application/json" } });
      }
    }

    if (url.pathname === "/api/delete-post" && request.method === "POST") {
      try {
        const { postId, sender } = await request.json();
        await env.DB.prepare("UPDATE posts SET is_deleted = 1 WHERE id = ? AND sender = ?").bind(postId, sender).run();
        return new Response(JSON.stringify({ success: true }));
      } catch (error) {
        return new Response(JSON.stringify({ error: error.message }), { status: 500 });
      }
    }

    if (url.pathname === "/api/post-like" && request.method === "POST") {
      try {
        const { postId, username } = await request.json();
        const existing = await env.DB.prepare("SELECT 1 FROM post_likes WHERE post_id = ? AND username = ?").bind(postId, username).first();
        if (existing) {
          await env.DB.prepare("DELETE FROM post_likes WHERE post_id = ? AND username = ?").bind(postId, username).run();
          await env.DB.prepare("UPDATE posts SET likes_count = MAX(0, likes_count - 1) WHERE id = ?").bind(postId).run();
          return new Response(JSON.stringify({ liked: false }));
        } else {
          await env.DB.prepare("INSERT INTO post_likes (post_id, username) VALUES (?, ?)").bind(postId, username).run();
          await env.DB.prepare("UPDATE posts SET likes_count = likes_count + 1 WHERE id = ?").bind(postId).run();
          return new Response(JSON.stringify({ liked: true }));
        }
      } catch (error) {
        return new Response(JSON.stringify({ error: error.message }), { status: 500 });
      }
    }

    if (url.pathname === "/api/post-repost" && request.method === "POST") {
      try {
        const { postId, username } = await request.json();
        const existing = await env.DB.prepare("SELECT 1 FROM post_reposts WHERE post_id = ? AND username = ?").bind(postId, username).first();
        if (existing) {
          await env.DB.prepare("DELETE FROM post_reposts WHERE post_id = ? AND username = ?").bind(postId, username).run();
          await env.DB.prepare("UPDATE posts SET reposts_count = MAX(0, reposts_count - 1) WHERE id = ?").bind(postId).run();
          return new Response(JSON.stringify({ reposted: false }));
        } else {
          await env.DB.prepare("INSERT INTO post_reposts (post_id, username) VALUES (?, ?)").bind(postId, username).run();
          await env.DB.prepare("UPDATE posts SET reposts_count = reposts_count + 1 WHERE id = ?").bind(postId).run();
          return new Response(JSON.stringify({ reposted: true }));
        }
      } catch (error) {
        return new Response(JSON.stringify({ error: error.message }), { status: 500 });
      }
    }

    if (url.pathname === "/api/post-view" && request.method === "POST") {
      try {
        const { postId, username } = await request.json();
        const exists = await env.DB.prepare("SELECT 1 FROM post_views WHERE post_id = ? AND username = ?").bind(postId, username).first();
        if (!exists) {
          await env.DB.prepare("INSERT INTO post_views (post_id, username) VALUES (?, ?)").bind(postId, username).run();
          await env.DB.prepare("UPDATE posts SET views_count = views_count + 1 WHERE id = ?").bind(postId).run();
          return new Response(JSON.stringify({ success: true, newView: true }));
        }
        return new Response(JSON.stringify({ success: true, newView: false }));
      } catch (error) {
        return new Response(JSON.stringify({ success: false, newView: false }));
      }
    }

    if (url.pathname === "/api/post-comments" && request.method === "GET") {
      try {
        const postId = url.searchParams.get('post_id');
        const result = await env.DB.prepare(
          "SELECT * FROM post_comments WHERE post_id = ? AND is_deleted = 0 ORDER BY created_at ASC"
        ).bind(postId).all();
        return new Response(JSON.stringify(result.results), { headers: { "Content-Type": "application/json" } });
      } catch (error) {
        return new Response(JSON.stringify([]), { headers: { "Content-Type": "application/json" } });
      }
    }

    if (url.pathname === "/api/post-comments" && request.method === "POST") {
      try {
        const { postId, sender, text } = await request.json();
        if (!text || !text.trim()) return new Response(JSON.stringify({ error: "متن الزامی است" }), { status: 400 });
        const result = await env.DB.prepare(
          "INSERT INTO post_comments (post_id, sender, text) VALUES (?, ?, ?) RETURNING *"
        ).bind(postId, sender, text.trim()).run();
        await env.DB.prepare("UPDATE posts SET comments_count = comments_count + 1 WHERE id = ?").bind(postId).run();
        return new Response(JSON.stringify(result.results[0]), { headers: { "Content-Type": "application/json" } });
      } catch (error) {
        return new Response(JSON.stringify({ error: error.message }), { status: 500, headers: { "Content-Type": "application/json" } });
      }
    }

    if (url.pathname === "/api/delete-post-comment" && request.method === "POST") {
      try {
        const { commentId, sender } = await request.json();
        await env.DB.prepare("UPDATE post_comments SET is_deleted = 1 WHERE id = ? AND sender = ?").bind(commentId, sender).run();
        return new Response(JSON.stringify({ success: true }));
      } catch (error) {
        return new Response(JSON.stringify({ error: error.message }), { status: 500 });
      }
    }

    return new Response("صفحه مورد نظر یافت نشد", { status: 404 });
  }
};

async function initDB(env) {
  try {
    await env.DB.prepare(`
      CREATE TABLE IF NOT EXISTS messages (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        text TEXT NOT NULL,
        sender TEXT NOT NULL,
        receiver TEXT NOT NULL,
        image_data TEXT DEFAULT NULL,
        reply_to_id INTEGER DEFAULT NULL,
        is_edited INTEGER DEFAULT 0,
        is_deleted INTEGER DEFAULT 0,
        is_read INTEGER DEFAULT 0,
        created_at DATETIME DEFAULT CURRENT_TIMESTAMP
      )
    `).run();

    const alterQueries = [
      "ALTER TABLE messages ADD COLUMN image_data TEXT DEFAULT NULL",
      "ALTER TABLE messages ADD COLUMN reply_to_id INTEGER DEFAULT NULL",
      "ALTER TABLE messages ADD COLUMN is_edited INTEGER DEFAULT 0",
      "ALTER TABLE messages ADD COLUMN is_read INTEGER DEFAULT 0",
      "ALTER TABLE messages ADD COLUMN voice_data TEXT DEFAULT NULL"
    ];
    for (const q of alterQueries) {
      try { await env.DB.prepare(q).run(); } catch (_) {}
    }
    
    await env.DB.prepare(`
      CREATE TABLE IF NOT EXISTS users (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        username TEXT UNIQUE,
        password TEXT NOT NULL,
        display_name TEXT,
        last_seen DATETIME DEFAULT CURRENT_TIMESTAMP,
        created_at DATETIME DEFAULT CURRENT_TIMESTAMP
      )
    `).run();
    
    try { await env.DB.prepare("ALTER TABLE users ADD COLUMN last_seen DATETIME DEFAULT CURRENT_TIMESTAMP").run(); } catch (_) {}
    try { await env.DB.prepare("ALTER TABLE users ADD COLUMN avatar TEXT DEFAULT NULL").run(); } catch (_) {}
    try { await env.DB.prepare("ALTER TABLE users ADD COLUMN bio TEXT DEFAULT NULL").run(); } catch (_) {}

    await env.DB.prepare(`
      CREATE TABLE IF NOT EXISTS group_messages (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        text TEXT NOT NULL,
        sender TEXT NOT NULL,
        image_data TEXT DEFAULT NULL,
        voice_data TEXT DEFAULT NULL,
        reply_to_id INTEGER DEFAULT NULL,
        is_edited INTEGER DEFAULT 0,
        is_deleted INTEGER DEFAULT 0,
        created_at DATETIME DEFAULT CURRENT_TIMESTAMP
      )
    `).run();
    try { await env.DB.prepare("ALTER TABLE group_messages ADD COLUMN voice_data TEXT DEFAULT NULL").run(); } catch (_) {}

    await env.DB.prepare(`
      CREATE TABLE IF NOT EXISTS posts (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        text TEXT DEFAULT '',
        sender TEXT NOT NULL,
        image_data TEXT DEFAULT NULL,
        likes_count INTEGER DEFAULT 0,
        comments_count INTEGER DEFAULT 0,
        reposts_count INTEGER DEFAULT 0,
        views_count INTEGER DEFAULT 0,
        is_deleted INTEGER DEFAULT 0,
        created_at DATETIME DEFAULT CURRENT_TIMESTAMP
      )
    `).run();
    await env.DB.prepare(`
      CREATE TABLE IF NOT EXISTS post_likes (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        post_id INTEGER NOT NULL,
        username TEXT NOT NULL,
        created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
        UNIQUE(post_id, username)
      )
    `).run();
    await env.DB.prepare(`
      CREATE TABLE IF NOT EXISTS post_reposts (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        post_id INTEGER NOT NULL,
        username TEXT NOT NULL,
        created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
        UNIQUE(post_id, username)
      )
    `).run();
    await env.DB.prepare(`
      CREATE TABLE IF NOT EXISTS post_views (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        post_id INTEGER NOT NULL,
        username TEXT NOT NULL,
        created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
        UNIQUE(post_id, username)
      )
    `).run();
    await env.DB.prepare(`
      CREATE TABLE IF NOT EXISTS post_comments (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        post_id INTEGER NOT NULL,
        sender TEXT NOT NULL,
        text TEXT NOT NULL,
        is_deleted INTEGER DEFAULT 0,
        created_at DATETIME DEFAULT CURRENT_TIMESTAMP
      )
    `).run();
    try { await env.DB.prepare("CREATE INDEX IF NOT EXISTS idx_posts_id ON posts(id)").run(); } catch (_) {}
    try { await env.DB.prepare("CREATE INDEX IF NOT EXISTS idx_post_likes ON post_likes(post_id, username)").run(); } catch (_) {}
    try { await env.DB.prepare("CREATE INDEX IF NOT EXISTS idx_post_comments ON post_comments(post_id)").run(); } catch (_) {}
    try { await env.DB.prepare("CREATE INDEX IF NOT EXISTS idx_post_views ON post_views(post_id, username)").run(); } catch (_) {}
    try { await env.DB.prepare("CREATE INDEX IF NOT EXISTS idx_msg_sender_receiver ON messages(sender, receiver)").run(); } catch (_) {}
    try { await env.DB.prepare("CREATE INDEX IF NOT EXISTS idx_msg_deleted ON messages(is_deleted)").run(); } catch (_) {}
    try { await env.DB.prepare("CREATE INDEX IF NOT EXISTS idx_msg_id ON messages(id)").run(); } catch (_) {}
    try { await env.DB.prepare("CREATE INDEX IF NOT EXISTS idx_msg_is_read ON messages(is_read, receiver)").run(); } catch (_) {}
    try { await env.DB.prepare("CREATE INDEX IF NOT EXISTS idx_grp_deleted ON group_messages(is_deleted)").run(); } catch (_) {}
    try { await env.DB.prepare("CREATE INDEX IF NOT EXISTS idx_grp_id ON group_messages(id)").run(); } catch (_) {}
    try { await env.DB.prepare("CREATE INDEX IF NOT EXISTS idx_users_username ON users(username)").run(); } catch (_) {}

    const defaultUsers = [
      { username: "mostafa", password: "1234", display_name: "مصطفی" },
      { username: "adamon", password: "4321", display_name: "ادمان" }
    ];
    
    for (const user of defaultUsers) {
      const existing = await env.DB.prepare("SELECT * FROM users WHERE username = ?").bind(user.username).first();
      if (!existing) {
        await env.DB.prepare("INSERT INTO users (username, password, display_name) VALUES (?, ?, ?)").bind(user.username, user.password, user.display_name).run();
      }
    }

  } catch (error) {
    console.error("خطا در دیتابیس:", error);
  }
}

function getHTML() {
  return `<!DOCTYPE html>
<html dir="rtl">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, viewport-fit=cover, user-scalable=no">
    <title>Peykak | پیام‌رسان</title>
    <style>
        * { margin: 0; padding: 0; box-sizing: border-box; }
        
        :root {
            --bg-gradient-start: #667eea;
            --bg-gradient-end: #764ba2;
            --card-bg: white;
            --text-primary: #333;
            --text-secondary: #666;
            --message-sent-start: #667eea;
            --message-sent-end: #764ba2;
            --message-received: white;
            --border-color: #ddd;
            --input-bg: white;
            --message-bg: #f5f5f5;
            --header-bg-start: #667eea;
            --header-bg-end: #764ba2;
            --reply-bg: rgba(0,0,0,0.08);
            --reply-bar: #667eea;
            --online-green: #4caf50;
            --offline-gray: #9e9e9e;
        }
        
        body.dark {
            --bg-gradient-start: #1a1a2e;
            --bg-gradient-end: #16213e;
            --card-bg: #1e1e2e;
            --text-primary: #eee;
            --text-secondary: #aaa;
            --message-sent-start: #0f3460;
            --message-sent-end: #16213e;
            --message-received: #2a2a3e;
            --border-color: #333;
            --input-bg: #2a2a3e;
            --message-bg: #252535;
            --header-bg-start: #0f3460;
            --header-bg-end: #16213e;
            --reply-bg: rgba(255,255,255,0.08);
            --reply-bar: #667eea;
            --online-green: #4caf50;
            --offline-gray: #6b6b6b;
        }
        
        body {
            font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Tahoma, sans-serif;
            background: linear-gradient(135deg, var(--bg-gradient-start), var(--bg-gradient-end));
            min-height: 100vh;
            min-height: 100dvh;
            display: flex;
            justify-content: center;
            align-items: center;
            transition: all 0.3s ease;
        }
        
        .container {
            width: 100%;
            max-width: 500px;
            height: 100vh;
            height: 100dvh;
            background: var(--card-bg);
            display: flex;
            flex-direction: column;
            overflow: hidden;
            box-shadow: 0 10px 40px rgba(0,0,0,0.2);
            transition: background 0.3s ease;
            position: relative;
        }
        
        @media (min-width: 768px) {
            .container {
                height: 90vh;
                border-radius: 30px;
                margin: 20px;
            }
        }
        
        .header {
            background: linear-gradient(135deg, var(--header-bg-start), var(--header-bg-end));
            color: white;
            padding: 12px 16px;
            padding-top: max(12px, env(safe-area-inset-top));
            display: flex;
            justify-content: space-between;
            align-items: center;
            flex-shrink: 0;
            z-index: 100;
            box-shadow: 0 2px 10px rgba(0,0,0,0.15);
        }
        
        .header h2 { font-size: 18px; }
        
        .login-box, .chat-box, .users-box, .settings-box {
            flex: 1;
            padding: 20px;
            display: flex;
            flex-direction: column;
            overflow-y: auto;
        }
        
        .input-group { margin-bottom: 15px; }
        
        .input-group label {
            display: block;
            margin-bottom: 8px;
            color: var(--text-primary);
            font-weight: 500;
            font-size: 14px;
        }
        
        input {
            width: 100%;
            padding: 12px;
            border: 2px solid var(--border-color);
            border-radius: 12px;
            font-size: 14px;
            background: var(--input-bg);
            color: var(--text-primary);
            transition: border-color 0.3s;
        }
        
        input:focus { outline: none; border-color: #667eea; }
        
        button {
            width: 100%;
            padding: 12px;
            background: linear-gradient(135deg, #667eea, #764ba2);
            color: white;
            border: none;
            border-radius: 12px;
            font-size: 16px;
            font-weight: 600;
            cursor: pointer;
            transition: transform 0.2s;
        }
        
        button:active { transform: scale(0.98); }
        
        .messages {
            flex: 1;
            overflow-y: auto;
            overflow-x: hidden;
            padding: 12px 12px 8px;
            background: var(--message-bg);
            display: flex;
            flex-direction: column;
            gap: 6px;
            overscroll-behavior: contain;
            -webkit-overflow-scrolling: touch;
            will-change: scroll-position;
        }
        
        .input-area {
            display: flex;
            padding: 10px 12px;
            padding-bottom: max(10px, env(safe-area-inset-bottom));
            gap: 8px;
            background: var(--card-bg);
            border-top: 1px solid var(--border-color);
            flex-shrink: 0;
            align-items: center;
        }
        
        .message {
            padding: 10px 14px;
            border-radius: 18px;
            max-width: 85%;
            word-wrap: break-word;
            position: relative;
            transition: transform 0.2s ease, background 0.2s ease;
            cursor: pointer;
            touch-action: pan-y pinch-zoom;
        }
        
        .message.swiping-right {
            transform: translateX(10px);
            background: var(--reply-bg);
        }
        
        .sent {
            background: linear-gradient(135deg, var(--message-sent-start), var(--message-sent-end));
            color: white;
            align-self: flex-end;
            border-bottom-right-radius: 4px;
        }
        
        .received {
            background: var(--message-received);
            color: var(--text-primary);
            align-self: flex-start;
            border-bottom-left-radius: 4px;
            border: 1px solid var(--border-color);
        }
        
        .sender { font-size: 10px; margin-bottom: 4px; opacity: 0.7; }
        .time { font-size: 9px; margin-top: 4px; opacity: 0.6; text-align: left; }
        .edited-label { font-size: 9px; opacity: 0.5; margin-right: 4px; font-style: italic; }
        .read-status { font-size: 8px; margin-left: 4px; opacity: 0.6; }
        
        .reply-preview {
            background: var(--reply-bg);
            border-right: 3px solid var(--reply-bar);
            border-radius: 8px;
            padding: 6px 10px;
            margin-bottom: 6px;
            font-size: 11px;
            opacity: 0.85;
            cursor: pointer;
        }
        .reply-preview .reply-sender { font-weight: bold; margin-bottom: 2px; }
        .reply-preview .reply-text { overflow: hidden; white-space: nowrap; text-overflow: ellipsis; max-width: 200px; }
        
        .reply-bar {
            display: flex;
            align-items: center;
            justify-content: space-between;
            padding: 8px 14px;
            background: var(--message-bg);
            border-top: 1px solid var(--border-color);
            border-bottom: 1px solid var(--border-color);
            font-size: 12px;
            color: var(--text-secondary);
            gap: 8px;
            flex-shrink: 0;
        }
        .reply-bar-content { display: flex; align-items: center; gap: 8px; flex: 1; overflow: hidden; }
        .reply-bar-line { width: 3px; height: 30px; border-radius: 3px; background: #667eea; flex-shrink: 0; }
        .reply-cancel-btn { background: none; border: none; cursor: pointer; font-size: 16px; color: var(--text-secondary); padding: 0 4px; width: auto; }

        .msg-image { max-width: 200px; max-height: 200px; border-radius: 10px; margin-top: 6px; cursor: pointer; display: block; }

        .image-preview-bar {
            display: flex; align-items: center; padding: 8px 14px; gap: 10px;
            background: var(--message-bg); border-top: 1px solid var(--border-color); flex-shrink: 0;
        }
        .image-preview-bar img { height: 50px; border-radius: 8px; border: 2px solid #667eea; }
        .remove-img-btn { background: none; border: none; cursor: pointer; font-size: 18px; color: #f44336; width: auto; padding: 0; }

        .img-modal { position: fixed; inset: 0; background: rgba(0,0,0,0.85); display: flex; align-items: center; justify-content: center; z-index: 1000; cursor: zoom-out; }
        .img-modal img { max-width: 95vw; max-height: 90vh; border-radius: 12px; }

        .message-actions { position: absolute; top: 2px; left: 5px; display: none; gap: 4px; }
        .message:hover .message-actions { display: flex; }
        .action-btn { background: rgba(0,0,0,0.5); border: none; border-radius: 10px; padding: 2px 6px; color: white; cursor: pointer; font-size: 10px; width: auto; }

        .edit-area { display: flex; flex-direction: column; gap: 5px; margin-top: 4px; }
        .edit-area textarea { width: 100%; padding: 6px 10px; border-radius: 10px; border: 1px solid rgba(255,255,255,0.4); background: rgba(255,255,255,0.15); color: white; font-size: 13px; resize: none; font-family: inherit; }
        .received .edit-area textarea { background: var(--input-bg); color: var(--text-primary); border-color: var(--border-color); }
        .edit-buttons { display: flex; gap: 5px; }
        .edit-save-btn, .edit-cancel-btn { width: auto; padding: 4px 10px; font-size: 11px; border-radius: 8px; }
        
        .input-area input { flex: 1; margin: 0; border-radius: 25px; }
        .input-area button { width: auto; padding: 12px 16px; border-radius: 25px; }

        .upload-btn { width: auto; padding: 8px 10px; border-radius: 25px; background: var(--message-bg); color: var(--text-secondary); border: 1px solid var(--border-color); font-size: 14px; line-height: 1; cursor: pointer; }
        #fileInput { display: none; }
        
        .last-seen-text { font-size: 10px; color: var(--text-secondary); margin-top: 2px; }
        .error { color: #f44336; text-align: center; margin-top: 10px; font-size: 12px; }
        .success { color: #4caf50; text-align: center; margin-top: 10px; font-size: 12px; }
        
        .status { font-size: 10px; text-align: center; padding: 6px; background: var(--message-bg); color: #0095f6; flex-shrink: 0; }
        
        .icon-btn { background: rgba(255,255,255,0.2); border: none; padding: 0; border-radius: 50%; cursor: pointer; font-size: 10px; color: white; width: 20px; height: 20px; display: flex; align-items: center; justify-content: center; }
        .icon-btn:active { transform: scale(0.95); }

        .user-card { background: var(--message-bg); border: 2px solid var(--border-color); border-radius: 15px; padding: 12px; margin-bottom: 10px; cursor: pointer; transition: all 0.2s; display: flex; align-items: center; gap: 12px; position: relative; }
        .user-card:active { transform: scale(0.98); }

        .unread-badge {
            min-width: 20px;
            height: 20px;
            background: linear-gradient(135deg, #667eea, #764ba2);
            color: white;
            border-radius: 10px;
            font-size: 11px;
            font-weight: 700;
            display: flex;
            align-items: center;
            justify-content: center;
            padding: 0 5px;
            flex-shrink: 0;
            margin-right: auto;
            animation: badgePop 0.3s cubic-bezier(0.34,1.56,0.64,1);
        }
        @keyframes badgePop {
            from { transform: scale(0); opacity: 0; }
            to   { transform: scale(1); opacity: 1; }
        }
        .user-card.has-unread {
            border-color: #667eea;
            background: linear-gradient(135deg, rgba(102,126,234,0.07), rgba(118,75,162,0.07));
        }

        .user-emoji { font-size: 30px; }
        .user-name { font-size: 15px; font-weight: 600; color: var(--text-primary); }
        
        .back-btn, .settings-btn { background: rgba(255,255,255,0.2); border: none; padding: 4px 10px; border-radius: 16px; cursor: pointer; font-size: 14px; color: white; width: auto; flex-shrink: 0; }
        
        .tab-buttons { display: flex; gap: 10px; margin-bottom: 20px; }
        .tab-btn { flex: 1; padding: 10px; background: var(--message-bg); color: var(--text-primary); border: 1px solid var(--border-color); }
        .tab-btn.active { background: linear-gradient(135deg, #667eea, #764ba2); color: white; border: none; }
        
        .settings-menu { display: flex; flex-direction: column; gap: 15px; }
        .settings-item { background: var(--message-bg); border: 1px solid var(--border-color); border-radius: 15px; padding: 15px; cursor: pointer; transition: all 0.2s; }
        .settings-item:active { transform: scale(0.98); }
        .settings-item-title { font-size: 15px; font-weight: 600; color: var(--text-primary); }
        .settings-item-desc { font-size: 11px; color: var(--text-secondary); margin-top: 4px; }
        .danger-item { border-color: #f44336; }
        .danger-text { color: #f44336; }
        .admin-item { border-color: #e53935 !important; background: rgba(229,57,53,0.07) !important; }
        
        .messages::-webkit-scrollbar { width: 4px; }
        .messages::-webkit-scrollbar-track { background: var(--border-color); border-radius: 10px; }
        .messages::-webkit-scrollbar-thumb { background: #667eea; border-radius: 10px; }
        
        .date-separator { text-align: center; font-size: 10px; color: var(--text-secondary); margin: 10px 0; }
        .date-separator span { background: var(--message-bg); padding: 2px 10px; border-radius: 10px; }

        .group-card { border-color: #667eea !important; background: linear-gradient(135deg, rgba(102,126,234,0.1), rgba(118,75,162,0.1)) !important; }

        .avatar { width: 44px; height: 44px; border-radius: 50%; object-fit: cover; border: 2px solid var(--border-color); flex-shrink: 0; }
        .avatar-placeholder { width: 44px; height: 44px; border-radius: 50%; background: linear-gradient(135deg, #667eea, #764ba2); display: flex; align-items: center; justify-content: center; font-size: 20px; flex-shrink: 0; color: white; font-weight: 600; }
        .avatar-large { width: 80px; height: 80px; border-radius: 50%; object-fit: cover; border: 3px solid #667eea; display: block; margin: 0 auto 10px; }
        .avatar-large-placeholder { width: 80px; height: 80px; border-radius: 50%; background: linear-gradient(135deg, #667eea, #764ba2); display: flex; align-items: center; justify-content: center; font-size: 36px; margin: 0 auto 10px; color: white; }
        .avatar-upload-area { display: flex; flex-direction: column; align-items: center; gap: 10px; padding: 20px; border: 2px dashed var(--border-color); border-radius: 16px; margin-bottom: 16px; cursor: pointer; }
        .msg-avatar { width: 26px; height: 26px; border-radius: 50%; object-fit: cover; border: 1px solid var(--border-color); flex-shrink: 0; }
        .msg-avatar-placeholder { width: 26px; height: 26px; border-radius: 50%; background: linear-gradient(135deg, #667eea, #764ba2); display: flex; align-items: center; justify-content: center; font-size: 12px; flex-shrink: 0; color: white; }
        .message-row { display: flex; align-items: flex-end; gap: 6px; }
        .message-row.sent-row { flex-direction: row-reverse; }

        .profile-modal { position: fixed; inset: 0; background: rgba(0,0,0,0.6); display: flex; align-items: center; justify-content: center; z-index: 1001; padding: 20px; }
        .profile-card { background: var(--card-bg); border-radius: 24px; padding: 28px 24px; width: 100%; max-width: 320px; text-align: center; box-shadow: 0 20px 60px rgba(0,0,0,0.3); position: relative; }
        .profile-card .close-btn { position: absolute; top: 12px; left: 14px; background: none; border: none; font-size: 20px; cursor: pointer; color: var(--text-secondary); width: auto; padding: 4px 8px; }
        .profile-card img.profile-big-avatar { width: 90px; height: 90px; border-radius: 50%; object-fit: cover; border: 3px solid #667eea; margin-bottom: 12px; cursor: pointer; }
        .profile-big-placeholder { width: 90px; height: 90px; border-radius: 50%; background: linear-gradient(135deg, #667eea, #764ba2); display: flex; align-items: center; justify-content: center; font-size: 40px; color: white; margin: 0 auto 12px; }
        .profile-card .profile-name { font-size: 18px; font-weight: 700; color: var(--text-primary); margin-bottom: 4px; }
        .profile-card .profile-username { font-size: 12px; color: var(--text-secondary); margin-bottom: 10px; }
        .profile-card .profile-bio { font-size: 13px; color: var(--text-secondary); background: var(--message-bg); border-radius: 12px; padding: 10px 14px; margin-bottom: 14px; text-align: right; line-height: 1.6; }
        .profile-card .profile-status { font-size: 12px; margin-bottom: 16px; display: inline-flex; align-items: center; gap: 6px; background: var(--message-bg); padding: 6px 12px; border-radius: 20px; }
        .status-dot { display: inline-block; width: 10px; height: 10px; border-radius: 50%; margin-left: 6px; }
        .status-dot.online { background-color: var(--online-green); box-shadow: 0 0 6px var(--online-green); }
        .status-dot.offline { background-color: var(--offline-gray); }

        .swipe-indicator {
            position: fixed;
            bottom: 20px;
            left: 50%;
            transform: translateX(-50%);
            background: rgba(102,126,234,0.9);
            color: white;
            padding: 6px 12px;
            border-radius: 20px;
            font-size: 11px;
            z-index: 1000;
            pointer-events: none;
            opacity: 0;
            transition: opacity 0.3s ease;
            backdrop-filter: blur(10px);
            white-space: nowrap;
        }
        
        .swipe-indicator.show {
            opacity: 1;
        }
        
        .swipe-progress {
            position: fixed;
            top: 0;
            left: 0;
            width: 0%;
            height: 3px;
            background: linear-gradient(90deg, #667eea, #764ba2);
            z-index: 10000;
            transition: width 0.05s linear;
            pointer-events: none;
        }
        
        .page-transition {
            animation: pageSlide 0.3s cubic-bezier(0.34, 1.2, 0.64, 1);
        }
        
        @keyframes pageSlide {
            from {
                opacity: 0.5;
                transform: translateX(20px);
            }
            to {
                opacity: 1;
                transform: translateX(0);
            }
        }
        
        .back-swipe-area {
            position: fixed;
            left: 0;
            top: 0;
            width: 15px;
            height: 100%;
            z-index: 50;
            pointer-events: none;
        }

        .toast-notif { position: fixed; top: 16px; left: 50%; transform: translateX(-50%) translateY(-80px); background: linear-gradient(135deg, #667eea, #764ba2); color: white; padding: 10px 16px; border-radius: 20px; font-size: 13px; box-shadow: 0 4px 20px rgba(0,0,0,0.25); z-index: 9999; display: flex; align-items: center; gap: 8px; max-width: 320px; cursor: pointer; transition: transform 0.35s cubic-bezier(0.34,1.56,0.64,1); white-space: nowrap; overflow: hidden; text-overflow: ellipsis; }
        .toast-notif.show { transform: translateX(-50%) translateY(0); }
        .toast-notif .toast-avatar { font-size: 18px; flex-shrink: 0; }
        .toast-notif .toast-body { display: flex; flex-direction: column; overflow: hidden; }
        .toast-notif .toast-sender { font-weight: 700; font-size: 12px; }
        .toast-notif .toast-text { font-size: 11px; opacity: 0.9; overflow: hidden; text-overflow: ellipsis; white-space: nowrap; max-width: 220px; }

        .admin-panel-header { display: flex; align-items: center; gap: 10px; background: linear-gradient(135deg, #e53935, #b71c1c); color: white; padding: 12px 16px; border-radius: 14px; margin-bottom: 16px; }
        .admin-panel-header .admin-icon { font-size: 28px; }
        .admin-panel-header .admin-title { font-size: 16px; font-weight: 700; }
        .admin-panel-header .admin-subtitle { font-size: 11px; opacity: 0.85; margin-top: 2px; }
        .admin-user-row { background: var(--message-bg); border: 1px solid var(--border-color); border-radius: 14px; padding: 12px 14px; margin-bottom: 10px; display: flex; align-items: center; gap: 12px; }
        .admin-user-info { flex: 1; overflow: hidden; }
        .admin-user-name { font-size: 14px; font-weight: 600; color: var(--text-primary); }
        .admin-user-meta { font-size: 11px; color: var(--text-secondary); margin-top: 3px; }
        .admin-delete-btn { width: auto; padding: 7px 14px; font-size: 12px; border-radius: 10px; background: linear-gradient(135deg, #e53935, #b71c1c); flex-shrink: 0; border: none; color: white; cursor: pointer; font-weight: 600; }
        .admin-self-badge { font-size: 11px; color: #667eea; padding: 6px 10px; background: rgba(102,126,234,0.12); border-radius: 10px; font-weight: 600; flex-shrink: 0; }

        .main-tabs {
            display: flex;
            background: linear-gradient(135deg, var(--header-bg-start), var(--header-bg-end));
            flex-shrink: 0;
            border-bottom: 1px solid rgba(255,255,255,0.15);
        }
        .main-tab-btn {
            flex: 1;
            padding: 10px 6px;
            background: transparent;
            color: rgba(255,255,255,0.6);
            border: none;
            border-radius: 0;
            font-size: 13px;
            font-weight: 600;
            cursor: pointer;
            border-bottom: 3px solid transparent;
            transition: all 0.2s;
            width: auto;
        }
        .main-tab-btn.active {
            color: white;
            border-bottom-color: white;
            background: transparent;
        }

        .compose-box {
            padding: 12px 14px;
            border-bottom: 6px solid var(--message-bg);
            display: flex;
            gap: 10px;
            align-items: flex-start;
            background: var(--card-bg);
            flex-shrink: 0;
        }
        .compose-box textarea {
            flex: 1;
            border: none;
            outline: none;
            background: transparent;
            color: var(--text-primary);
            font-size: 15px;
            resize: none;
            font-family: inherit;
            min-height: 52px;
            max-height: 120px;
            line-height: 1.5;
        }
        .compose-box textarea::placeholder { color: var(--text-secondary); }
        .compose-actions {
            display: flex;
            align-items: center;
            justify-content: space-between;
            padding: 6px 14px 10px;
            border-bottom: 6px solid var(--message-bg);
            background: var(--card-bg);
            flex-shrink: 0;
        }
        .compose-icon-btn {
            background: none;
            border: none;
            cursor: pointer;
            font-size: 18px;
            color: #667eea;
            padding: 4px 8px;
            border-radius: 50%;
            width: auto;
            transition: background 0.2s;
        }
        .compose-icon-btn:hover { background: rgba(102,126,234,0.1); }
        .compose-submit-btn {
            background: linear-gradient(135deg, #667eea, #764ba2);
            color: white;
            border: none;
            border-radius: 20px;
            padding: 8px 20px;
            font-size: 13px;
            font-weight: 700;
            cursor: pointer;
            width: auto;
            transition: opacity 0.2s;
        }
        .compose-submit-btn:disabled { opacity: 0.4; cursor: default; }
        .compose-img-preview {
            position: relative;
            display: inline-block;
            margin: 0 14px 8px;
        }
        .compose-img-preview img { max-height: 160px; border-radius: 12px; border: 2px solid var(--border-color); display: block; }
        .compose-img-remove {
            position: absolute; top: -6px; right: -6px;
            background: #f44336; color: white; border: none; border-radius: 50%;
            width: 22px; height: 22px; cursor: pointer; font-size: 12px;
            display: flex; align-items: center; justify-content: center;
        }

        .post-feed {
            flex: 1;
            overflow-y: auto;
            background: var(--message-bg);
            -webkit-overflow-scrolling: touch;
        }
        .post-card {
            background: var(--card-bg);
            border-bottom: 6px solid var(--message-bg);
            padding: 14px 14px 0;
        }
        .post-header {
            display: flex;
            align-items: center;
            gap: 10px;
            margin-bottom: 10px;
        }
        .post-user-info { flex: 1; overflow: hidden; }
        .post-display-name { font-size: 14px; font-weight: 700; color: var(--text-primary); }
        .post-username-time { font-size: 11px; color: var(--text-secondary); margin-top: 1px; }
        .post-text { font-size: 15px; color: var(--text-primary); line-height: 1.6; margin-bottom: 10px; white-space: pre-wrap; word-break: break-word; }
        .post-image { width: 100%; max-height: 320px; object-fit: cover; border-radius: 14px; margin-bottom: 10px; cursor: pointer; display: block; }
        .post-actions {
            display: flex;
            align-items: center;
            padding: 4px 0 10px;
            border-top: 1px solid var(--border-color);
            margin-top: 4px;
        }
        .post-action-btn {
            display: flex;
            align-items: center;
            gap: 5px;
            background: none;
            border: none;
            cursor: pointer;
            color: var(--text-secondary);
            font-size: 12px;
            padding: 7px 10px;
            border-radius: 20px;
            width: auto;
            transition: all 0.15s;
            font-weight: 500;
            flex: 1;
            justify-content: center;
        }
        .post-action-btn .icon { font-size: 16px; line-height: 1; }
        .post-action-btn:hover { background: var(--message-bg); }
        .post-action-btn.liked { color: #e0245e; }
        .post-action-btn.liked .icon { animation: heartPop 0.3s cubic-bezier(0.34,1.56,0.64,1); }
        .post-action-btn.reposted { color: #17bf63; }
        @keyframes heartPop {
            0% { transform: scale(1); }
            50% { transform: scale(1.5); }
            100% { transform: scale(1); }
        }
        .post-delete-btn {
            background: none; border: none; cursor: pointer;
            color: var(--text-secondary); font-size: 16px; padding: 4px 8px;
            border-radius: 50%; width: auto;
        }
        .post-delete-btn:hover { color: #f44336; background: rgba(244,67,54,0.1); }

        .comments-section {
            background: var(--message-bg);
            border-top: 1px solid var(--border-color);
            padding: 0;
            overflow: hidden;
            transition: max-height 0.3s ease;
        }
        .comment-item {
            display: flex;
            gap: 8px;
            padding: 10px 14px;
            border-bottom: 1px solid var(--border-color);
            background: var(--card-bg);
        }
        .comment-body { flex: 1; }
        .comment-sender { font-size: 12px; font-weight: 700; color: var(--text-primary); }
        .comment-text { font-size: 13px; color: var(--text-primary); margin-top: 2px; line-height: 1.5; word-break: break-word; }
        .comment-time { font-size: 10px; color: var(--text-secondary); margin-top: 3px; }
        .comment-del-btn { background: none; border: none; cursor: pointer; color: var(--text-secondary); font-size: 14px; width: auto; padding: 0 4px; align-self: flex-start; }
        .comment-input-row {
            display: flex;
            gap: 8px;
            padding: 10px 14px;
            background: var(--card-bg);
            align-items: center;
            border-top: 1px solid var(--border-color);
        }
        .comment-input-row input { flex: 1; padding: 8px 14px; border-radius: 20px; font-size: 13px; background: var(--message-bg); border: 1px solid var(--border-color); }
        .comment-send-btn { width: auto; padding: 8px 14px; border-radius: 20px; font-size: 13px; }

        .empty-feed { text-align: center; padding: 50px 20px; color: var(--text-secondary); font-size: 14px; }

        .hamburger-menu {
            position: fixed;
            inset: 0;
            z-index: 500;
        }
        .hamburger-overlay {
            position: absolute;
            inset: 0;
            background: rgba(0,0,0,0.45);
        }
        .hamburger-panel {
            position: absolute;
            top: 0;
            right: 0;
            width: min(280px, 82vw);
            height: 100%;
            background: var(--card-bg);
            box-shadow: -4px 0 30px rgba(0,0,0,0.25);
            display: flex;
            flex-direction: column;
            animation: slideInPanel 0.25s cubic-bezier(0.34,1.26,0.64,1);
            overflow-y: auto;
        }
        @keyframes slideInPanel {
            from { transform: translateX(100%); opacity: 0.5; }
            to   { transform: translateX(0); opacity: 1; }
        }
        .hamburger-header {
            display: flex;
            align-items: center;
            gap: 12px;
            padding: 20px 16px 16px;
            background: linear-gradient(135deg, var(--header-bg-start), var(--header-bg-end));
            flex-shrink: 0;
        }
        .hamburger-header .avatar,
        .hamburger-header .avatar-placeholder {
            width: 48px;
            height: 48px;
            font-size: 22px;
            flex-shrink: 0;
            border-color: rgba(255,255,255,0.4);
        }
        .hamburger-name { font-weight: 700; font-size: 15px; color: white; }
        .hamburger-username { font-size: 11px; color: rgba(255,255,255,0.75); margin-top: 3px; }
        .hamburger-divider {
            height: 1px;
            background: var(--border-color);
            margin: 4px 0;
            flex-shrink: 0;
        }
        .hamburger-item {
            display: flex;
            align-items: center;
            gap: 14px;
            padding: 15px 20px;
            background: none;
            border: none;
            border-radius: 0;
            width: 100%;
            text-align: right;
            cursor: pointer;
            transition: background 0.15s;
            color: var(--text-primary);
        }
        .hamburger-item:hover,
        .hamburger-item:active { background: var(--message-bg); }
        .hamburger-icon { font-size: 20px; flex-shrink: 0; }
        .hamburger-label { font-size: 14px; font-weight: 500; flex: 1; text-align: right; }
        .danger-menu-item .hamburger-label { color: #f44336; }
        .hamburger-btn {
            width: 34px !important;
            height: 34px !important;
            font-size: 18px !important;
            border-radius: 10px !important;
        }
    </style>
</head>
<body>
    <div class="container" id="app"></div>
    <div class="swipe-indicator" id="swipeIndicator"></div>
    <div class="swipe-progress" id="swipeProgress"></div>
    <div class="back-swipe-area" id="backSwipeArea"></div>
    <input type="file" id="fileInput" accept="image/*">

    <script>
        let currentUser = null;
        let currentChatUser = null;
        let refreshInterval = null;
        let activeTab = 'login';
        let replyingTo = null;
        let pendingImage = null;
        let allMessages = [];
        let lastSeenInterval = null;
        let notificationPermission = false;
        let usersRefreshInterval = null;
        
        let lastMsgId = 0;
        let isRendering = false;
        let pendingRender = null;

        let unreadCounts = {};
        
        let currentMainTab = 'messages';
        let userOnlineStatus = {};
        let onlineStatusInterval = null;
        
        let allPosts = [];
        let postRefreshInterval = null;
        let pendingPostImage = null;
        let openCommentPostId = null;
        const localLikes = {};
        const localReposts = {};

        const avatarCache = {};
        const bioCache = {};
        const avatarCacheTime = {};

        const LANGS = {
            fa: {
                dir: 'rtl',
                appName: 'Peykak',
                login: 'ورود',
                register: 'ثبت‌نام',
                username: 'نام کاربری',
                password: 'رمز عبور',
                usernamePlaceholder: 'نام کاربری',
                passwordPlaceholder: 'رمز عبور',
                loginBtn: 'ورود به Peykak',
                regUsernamePlaceholder: 'نام کاربری (فقط انگلیسی)',
                regPasswordPlaceholder: 'رمز عبور',
                displayName: 'نام نمایشی',
                displayNamePlaceholder: 'نامی که دیگران می‌بینند',
                registerBtn: 'ثبت‌نام',
                fillFields: 'لطفاً نام کاربری و رمز عبور را وارد کنید',
                registerSuccess: '✅ ثبت‌نام موفق! اکنون وارد شوید',
                serverError: 'خطا در ارتباط با سرور',
                greeting: (name) => \`سلام، \${name}\`,
                tabMessages: '💬 پیام‌ها',
                tabTimeline: '📰 تایم‌لاین',
                selectContact: 'انتخاب مخاطب',
                publicGroup: 'گروه عمومی',
                publicGroupSub: 'همه کاربران',
                noOtherUsers: 'هیچ کاربر دیگری وجود ندارد',
                statusNew: 'تازه‌وارد',
                statusOnline: 'آنلاین',
                statusOffline: 'آفلاین',
                statusUnknown: 'نامشخص',
                lastSeenToday: (t) => \`آخرین بازدید: امروز \${t}\`,
                lastSeenYesterday: (t) => \`آخرین بازدید: دیروز \${t}\`,
                lastSeenDate: (d, t) => \`آخرین بازدید: \${d} \${t}\`,
                messagePlaceholder: 'پیام...',
                groupMessagePlaceholder: 'پیام به گروه...',
                deleteConfirm: 'این پیام حذف شود؟',
                edited: 'ویرایش‌شده',
                settings: 'تنظیمات',
                changeAvatar: 'تغییر عکس پروفایل',
                changeAvatarHint: 'برای انتخاب عکس کلیک کنید',
                bio: 'بیوگرافی',
                bioPlaceholder: 'درباره خودت بنویس...',
                saveBio: 'ذخیره بیوگرافی',
                bioSaved: '✅ بیوگرافی ذخیره شد',
                changeDisplayName: 'نام نمایشی',
                saveDisplayName: 'ذخیره',
                displayNameSaved: '✅ نام نمایشی تغییر کرد',
                displayNameEmpty: 'نام نمایشی نمی‌تواند خالی باشد',
                notifications: '🔔 فعال‌سازی اعلان',
                notificationsDesc: 'دریافت اعلان پیام‌های جدید',
                notifGranted: '✅ اعلان‌ها فعال شد!',
                notifDenied: '❌ لطفاً دسترسی اعلان را در مرورگر فعال کنید',
                language: '🌐 زبان',
                languageDesc: 'تغییر زبان نمایش',
                changePassword: '🔐 تغییر رمز عبور',
                changePasswordDesc: 'رمز عبور حساب خود را تغییر دهید',
                currentPassword: 'رمز عبور فعلی',
                newPassword: 'رمز عبور جدید',
                confirmPassword: 'تکرار رمز جدید',
                currentPasswordPlaceholder: 'رمز فعلی را وارد کنید',
                newPasswordPlaceholder: 'رمز جدید',
                confirmPasswordPlaceholder: 'تکرار رمز جدید',
                changePasswordBtn: 'تغییر رمز',
                passwordChanged: '✅ رمز عبور با موفقیت تغییر کرد',
                wrongPassword: 'رمز عبور فعلی اشتباه است',
                passwordMismatch: 'رمزهای جدید مطابقت ندارند',
                deleteAccount: '🗑️ حذف حساب',
                deleteAccountDesc: 'حساب و تمام پیام‌های شما حذف می‌شود',
                deleteWarning: '⚠️ این عمل غیرقابل بازگشت است!',
                deletePasswordPlaceholder: 'رمز عبور خود را وارد کنید',
                deleteAccountBtn: 'حذف حساب',
                deleteConfirmAlert: 'مطمئن هستید؟ این عمل برگشت‌پذیر نیست و تمام پیام‌های شما حذف خواهد شد.',
                adminPanel: '👑 پنل مدیریت',
                adminPanelDesc: 'مدیریت و حذف کاربران',
                adminTitle: 'مدیریت کاربران',
                adminSubtitle: 'حذف کاربر باعث پاک شدن تمام پیام‌هایش می‌شود',
                adminDeleteBtn: '🗑️ حذف',
                adminSelfBadge: '👑 شما',
                adminDeleteConfirm: (name, uname) => \`⚠️ مطمئن هستید؟\n\nکاربر "\${name}" (@\${uname}) و تمام پیام‌هایش حذف می‌شوند.\n\nاین عمل برگشت‌پذیر نیست!\`,
                adminDeleteSuccess: (name) => \`\${name} و تمام پیام‌هایش حذف شدند\`,
                avatarSaved: '✅ عکس پروفایل ذخیره شد',
                avatarTooBig: 'حجم عکس نباید بیشتر از ۲ مگابایت باشد',
                onlyImage: 'فقط فایل تصویری انتخاب کنید',
                whatsOnYourMind: 'چی تو ذهنته؟ بنویس...',
                post: 'پست',
                loading: '⏳ در حال بارگذاری...',
                emptyFeed: '✨ اولین پست را بنویس!',
                emptyChat: '✨ اولین پیام را بفرست ✨',
                emptyGroup: '✨ اولین پیام گروه را بفرست ✨',
                deletePostConfirm: 'این پست حذف شود؟',
                comment: 'کامنت',
                commentPlaceholder: 'کامنت بنویس...',
                sendComment: 'ارسال',
                noComments: 'هنوز کامنتی نیست',
                deleteCommentConfirm: 'کامنت حذف شود؟',
                justNow: 'همین الان',
                minutesAgo: (n) => \`\${n} دقیقه پیش\`,
                hoursAgo: (n) => \`\${n} ساعت پیش\`,
                daysAgo: (n) => \`\${n} روز پیش\`,
                today: 'امروز',
                yesterday: 'دیروز',
                usernameTaken: 'این نام کاربری قبلاً ثبت شده است',
                wrongCredentials: 'نام کاربری یا رمز عبور اشتباه است',
                selectLanguage: 'انتخاب زبان',
                langFa: '🇮🇷 فارسی',
                langEn: '🇺🇸 English',
                darkMode: 'حالت تاریک / روشن',
                logout: 'خروج از حساب',
                swipeBack: '🔙 برگشت',
                swipeReply: '↩️ پاسخ',
                swipeToMessages: '📋 رفتن به پیام‌ها',
                swipeToTimeline: '📰 رفتن به تایم‌لاین',
                messageActions: '📋 منوی پیام',
                copyText: '📋 کپی متن',
                copied: '✅ متن کپی شد'
            },
            en: {
                dir: 'ltr',
                appName: 'Peykak',
                login: 'Login',
                register: 'Sign Up',
                username: 'Username',
                password: 'Password',
                usernamePlaceholder: 'Username',
                passwordPlaceholder: 'Password',
                loginBtn: 'Sign in to Peykak',
                regUsernamePlaceholder: 'Username (English only)',
                regPasswordPlaceholder: 'Password',
                displayName: 'Display Name',
                displayNamePlaceholder: 'Name others will see',
                registerBtn: 'Create Account',
                fillFields: 'Please enter username and password',
                registerSuccess: '✅ Account created! Please sign in',
                serverError: 'Server connection error',
                greeting: (name) => \`Hey, \${name}\`,
                tabMessages: '💬 Messages',
                tabTimeline: '📰 Timeline',
                selectContact: 'Select Contact',
                publicGroup: 'Public Group',
                publicGroupSub: 'All users',
                noOtherUsers: 'No other users found',
                statusNew: 'New',
                statusOnline: 'Online',
                statusOffline: 'Offline',
                statusUnknown: 'Unknown',
                lastSeenToday: (t) => \`Last seen today at \${t}\`,
                lastSeenYesterday: (t) => \`Last seen yesterday at \${t}\`,
                lastSeenDate: (d, t) => \`Last seen \${d} at \${t}\`,
                messagePlaceholder: 'Message...',
                groupMessagePlaceholder: 'Message the group...',
                deleteConfirm: 'Delete this message?',
                edited: 'edited',
                settings: 'Settings',
                changeAvatar: 'Change Profile Photo',
                changeAvatarHint: 'Click to choose a photo',
                bio: 'Bio',
                bioPlaceholder: 'Tell us about yourself...',
                saveBio: 'Save Bio',
                bioSaved: '✅ Bio saved',
                changeDisplayName: 'Display Name',
                saveDisplayName: 'Save',
                displayNameSaved: '✅ Display name updated',
                displayNameEmpty: 'Display name cannot be empty',
                notifications: '🔔 Enable Notifications',
                notificationsDesc: 'Receive alerts for new messages',
                notifGranted: '✅ Notifications enabled!',
                notifDenied: '❌ Please allow notifications in your browser',
                language: '🌐 Language',
                languageDesc: 'Change display language',
                changePassword: '🔐 Change Password',
                changePasswordDesc: 'Update your account password',
                currentPassword: 'Current Password',
                newPassword: 'New Password',
                confirmPassword: 'Confirm New Password',
                currentPasswordPlaceholder: 'Enter current password',
                newPasswordPlaceholder: 'New password',
                confirmPasswordPlaceholder: 'Confirm new password',
                changePasswordBtn: 'Update Password',
                passwordChanged: '✅ Password updated successfully',
                wrongPassword: 'Current password is incorrect',
                passwordMismatch: 'New passwords do not match',
                deleteAccount: '🗑️ Delete Account',
                deleteAccountDesc: 'Permanently delete your account and all messages',
                deleteWarning: '⚠️ This action is irreversible!',
                deletePasswordPlaceholder: 'Enter your password to confirm',
                deleteAccountBtn: 'Delete My Account',
                deleteConfirmAlert: 'Are you sure? This cannot be undone and all your messages will be permanently deleted.',
                adminPanel: '👑 Admin Panel',
                adminPanelDesc: 'Manage and remove users',
                adminTitle: 'User Management',
                adminSubtitle: 'Deleting a user removes all their messages',
                adminDeleteBtn: '🗑️ Delete',
                adminSelfBadge: '👑 You',
                adminDeleteConfirm: (name, uname) => \`⚠️ Are you sure?\n\nUser "\${name}" (@\${uname}) and all their messages will be permanently deleted.\n\nThis cannot be undone!\`,
                adminDeleteSuccess: (name) => \`\${name} and all their messages were deleted\`,
                avatarSaved: '✅ Profile photo saved',
                avatarTooBig: 'Photo must be under 2 MB',
                onlyImage: 'Please select an image file only',
                whatsOnYourMind: "What's on your mind?",
                post: 'Post',
                loading: '⏳ Loading...',
                emptyFeed: '✨ Be the first to post!',
                emptyChat: '✨ Send the first message ✨',
                emptyGroup: '✨ Send the first group message ✨',
                deletePostConfirm: 'Delete this post?',
                comment: 'Comment',
                commentPlaceholder: 'Write a comment...',
                sendComment: 'Send',
                noComments: 'No comments yet',
                deleteCommentConfirm: 'Delete this comment?',
                justNow: 'Just now',
                minutesAgo: (n) => \`\${n}m ago\`,
                hoursAgo: (n) => \`\${n}h ago\`,
                daysAgo: (n) => \`\${n}d ago\`,
                today: 'Today',
                yesterday: 'Yesterday',
                usernameTaken: 'This username is already taken',
                wrongCredentials: 'Incorrect username or password',
                selectLanguage: 'Select Language',
                langFa: '🇮🇷 Persian',
                langEn: '🇺🇸 English',
                darkMode: 'Dark / Light Mode',
                logout: 'Sign Out',
                swipeBack: '🔙 Back',
                swipeReply: '↩️ Reply',
                swipeToMessages: '📋 Go to Messages',
                swipeToTimeline: '📰 Go to Timeline',
                messageActions: '📋 Message Menu',
                copyText: '📋 Copy Text',
                copied: '✅ Copied'
            }
        };

        let currentLang = localStorage.getItem('lang') || 'fa';
        function t() { return LANGS[currentLang]; }

        function setLang(lang) {
            currentLang = lang;
            localStorage.setItem('lang', lang);
            document.documentElement.dir = LANGS[lang].dir;
            document.documentElement.lang = lang;
        }

        document.documentElement.dir = LANGS[currentLang].dir;
        document.documentElement.lang = currentLang;

        if (localStorage.getItem('darkMode') === 'true') {
            document.body.classList.add('dark');
        }

        function toggleDarkMode() {
            document.body.classList.toggle('dark');
            localStorage.setItem('darkMode', document.body.classList.contains('dark'));
        }

        function toTehranTime(utcDateString) {
            if (!utcDateString) return null;
            const s = utcDateString.endsWith('Z') || utcDateString.includes('+') ? utcDateString : utcDateString.replace(' ', 'T') + 'Z';
            const utcDate = new Date(s);
            if (isNaN(utcDate)) return null;
            return new Date(utcDate.getTime() + (3.5 * 60 * 60 * 1000));
        }

        function isOnline(lastSeen) {
            if (!lastSeen) return false;
            const tehranLastSeen = toTehranTime(lastSeen);
            if (!tehranLastSeen) return false;
            const nowTehran = new Date(new Date().getTime() + (3.5 * 60 * 60 * 1000));
            const diffSeconds = (nowTehran - tehranLastSeen) / 1000;
            return diffSeconds < 120;
        }

        function getStatusText(lastSeen) {
            const L = t();
            if (!lastSeen) return L.statusNew;
            if (isOnline(lastSeen)) return L.statusOnline;
            const tehranTime = toTehranTime(lastSeen);
            if (!tehranTime) return L.statusUnknown;
            const nowTehran = new Date(new Date().getTime() + (3.5 * 60 * 60 * 1000));
            const todayTehran = new Date(nowTehran.getFullYear(), nowTehran.getMonth(), nowTehran.getDate());
            const lastSeenDate = new Date(tehranTime.getFullYear(), tehranTime.getMonth(), tehranTime.getDate());
            const hours = tehranTime.getHours().toString().padStart(2, '0');
            const minutes = tehranTime.getMinutes().toString().padStart(2, '0');
            const timeString = hours + ':' + minutes;
            if (lastSeenDate.getTime() === todayTehran.getTime()) {
                return L.lastSeenToday(timeString);
            } else {
                const diffDays = Math.floor((todayTehran - lastSeenDate) / (1000 * 60 * 60 * 24));
                if (diffDays === 1) return L.lastSeenYesterday(timeString);
                const dateStr = currentLang === 'fa'
                    ? tehranTime.toLocaleDateString('fa-IR')
                    : tehranTime.toLocaleDateString('en-US', { month: 'short', day: 'numeric' });
                return L.lastSeenDate(dateStr, timeString);
            }
        }
        
        function getStatusWithIcon(lastSeen) {
            const L = t();
            if (!lastSeen) return { text: L.statusNew, isOnline: false };
            const online = isOnline(lastSeen);
            if (online) return { text: L.statusOnline, isOnline: true };
            const tehranTime = toTehranTime(lastSeen);
            if (!tehranTime) return { text: L.statusUnknown, isOnline: false };
            const nowTehran = new Date(new Date().getTime() + (3.5 * 60 * 60 * 1000));
            const todayTehran = new Date(nowTehran.getFullYear(), nowTehran.getMonth(), nowTehran.getDate());
            const lastSeenDate = new Date(tehranTime.getFullYear(), tehranTime.getMonth(), tehranTime.getDate());
            const hours = tehranTime.getHours().toString().padStart(2, '0');
            const minutes = tehranTime.getMinutes().toString().padStart(2, '0');
            const timeString = hours + ':' + minutes;
            let text = '';
            if (lastSeenDate.getTime() === todayTehran.getTime()) {
                text = L.lastSeenToday(timeString);
            } else {
                const diffDays = Math.floor((todayTehran - lastSeenDate) / (1000 * 60 * 60 * 24));
                if (diffDays === 1) text = L.lastSeenYesterday(timeString);
                else {
                    const dateStr = currentLang === 'fa'
                        ? tehranTime.toLocaleDateString('fa-IR')
                        : tehranTime.toLocaleDateString('en-US', { month: 'short', day: 'numeric' });
                    text = L.lastSeenDate(dateStr, timeString);
                }
            }
            return { text, isOnline: false };
        }
        
        async function updateLastSeen() {
            if (currentUser) {
                await fetch('/api/update-last-seen', {
                    method: 'POST',
                    headers: { 'Content-Type': 'application/json' },
                    body: JSON.stringify({ username: currentUser.username })
                });
            }
        }

        async function fetchUnreadCounts() {
            if (!currentUser) return;
            try {
                const res = await fetch('/api/unread-counts?receiver=' + encodeURIComponent(currentUser.username));
                unreadCounts = await res.json();
            } catch(e) {}
        }
        
        async function requestNotificationPermission() {
            if (!('Notification' in window)) return false;
            const permission = await Notification.requestPermission();
            notificationPermission = permission === 'granted';
            if (notificationPermission) localStorage.setItem('notificationEnabled', 'true');
            return notificationPermission;
        }
        
        let toastTimeout = null;

        function sendNotification(title, body, onClick) {
            if (notificationPermission && document.hidden) {
                try {
                    const n = new Notification(title, { body, icon: '💬', tag: 'peykak-message', renotify: true, silent: false });
                    if (onClick) n.onclick = () => { window.focus(); onClick(); n.close(); };
                } catch(e) {}
            }
            showToast(title, body, onClick);
        }

        function showToast(title, body, onClick) {
            const old = document.getElementById('peykak-toast');
            if (old) old.remove();
            if (toastTimeout) clearTimeout(toastTimeout);
            const toast = document.createElement('div');
            toast.id = 'peykak-toast';
            toast.className = 'toast-notif';
            toast.innerHTML = \`<div class="toast-avatar">💬</div><div class="toast-body"><div class="toast-sender">\${escapeHtml(title)}</div><div class="toast-text">\${escapeHtml(body)}</div></div>\`;
            toast.onclick = () => { toast.remove(); if (onClick) onClick(); };
            document.body.appendChild(toast);
            requestAnimationFrame(() => { requestAnimationFrame(() => toast.classList.add('show')); });
            toastTimeout = setTimeout(() => {
                toast.classList.remove('show');
                setTimeout(() => { if (toast.parentNode) toast.remove(); }, 400);
            }, 4000);
        }

        function showSwipeIndicator(text, duration = 800) {
            const indicator = document.getElementById('swipeIndicator');
            if (!indicator) return;
            indicator.textContent = text;
            indicator.classList.add('show');
            setTimeout(() => {
                indicator.classList.remove('show');
            }, duration);
        }
        
        function updateSwipeProgress(percent) {
            const progress = document.getElementById('swipeProgress');
            if (!progress) return;
            progress.style.width = Math.min(100, Math.max(0, percent)) + '%';
            if (percent >= 100) {
                setTimeout(() => {
                    progress.style.width = '0%';
                }, 200);
            } else if (percent <= 0) {
                progress.style.width = '0%';
            }
        }
        
        function setupMainTabSwipes(container) {
            if (!container) return;
            
            let touchStartX = 0;
            let touchStartY = 0;
            let swipeDetected = false;
            
            const onTouchStart = (e) => {
                touchStartX = e.touches[0].clientX;
                touchStartY = e.touches[0].clientY;
                swipeDetected = false;
            };
            
            const onTouchMove = (e) => {
                if (swipeDetected) return;
                const deltaX = e.touches[0].clientX - touchStartX;
                const deltaY = e.touches[0].clientY - touchStartY;
                
                if (Math.abs(deltaX) > 30 && Math.abs(deltaX) > Math.abs(deltaY)) {
                    swipeDetected = true;
                    
                    if (deltaX > 0) {
                        if (currentMainTab !== 'messages') {
                            showSwipeIndicator(t().swipeToMessages);
                            currentMainTab = 'messages';
                            showUsersList('messages');
                        }
                    } else if (deltaX < 0) {
                        if (currentMainTab !== 'posts') {
                            showSwipeIndicator(t().swipeToTimeline);
                            currentMainTab = 'posts';
                            showUsersList('posts');
                        }
                    }
                }
            };
            
            container.addEventListener('touchstart', onTouchStart, { passive: true });
            container.addEventListener('touchmove', onTouchMove, { passive: false });
        }
        
        function setupChatBackSwipe(container, onBackCallback) {
            if (!container) return;
            
            let touchStartX = 0;
            let touchStartY = 0;
            let swipeProgress = 0;
            let isSwipingBack = false;
            
            const onTouchStart = (e) => {
                touchStartX = e.touches[0].clientX;
                touchStartY = e.touches[0].clientY;
                isSwipingBack = false;
                swipeProgress = 0;
                updateSwipeProgress(0);
            };
            
            const onTouchMove = (e) => {
                const deltaX = e.touches[0].clientX - touchStartX;
                const deltaY = e.touches[0].clientY - touchStartY;
                
                if (deltaX > 20 && Math.abs(deltaX) > Math.abs(deltaY) && !isSwipingBack) {
                    isSwipingBack = true;
                    swipeProgress = Math.min(100, (deltaX / 70) * 100);
                    updateSwipeProgress(swipeProgress);
                    
                    if (swipeProgress >= 100) {
                        showSwipeIndicator(t().swipeBack, 300);
                        if (onBackCallback) onBackCallback();
                        isSwipingBack = false;
                        updateSwipeProgress(0);
                    }
                } else if (isSwipingBack) {
                    swipeProgress = Math.min(100, (deltaX / 70) * 100);
                    updateSwipeProgress(swipeProgress);
                }
            };
            
            const onTouchEnd = () => {
                if (isSwipingBack) {
                    setTimeout(() => updateSwipeProgress(0), 100);
                }
                isSwipingBack = false;
            };
            
            container.addEventListener('touchstart', onTouchStart, { passive: true });
            container.addEventListener('touchmove', onTouchMove, { passive: false });
            container.addEventListener('touchend', onTouchEnd);
        }
        
        function setupMessageReplySwipe(messageElement, messageId, onReplyCallback) {
            if (!messageElement) return;
            
            let touchStartX = 0;
            let touchStartY = 0;
            let swipeProgress = 0;
            let isSwipingReply = false;
            
            const onTouchStart = (e) => {
                touchStartX = e.touches[0].clientX;
                touchStartY = e.touches[0].clientY;
                isSwipingReply = false;
                swipeProgress = 0;
                messageElement.classList.remove('swiping-right');
            };
            
            const onTouchMove = (e) => {
                const deltaX = e.touches[0].clientX - touchStartX;
                const deltaY = e.touches[0].clientY - touchStartY;
                
                if (deltaX > 15 && Math.abs(deltaX) > Math.abs(deltaY) && !isSwipingReply) {
                    e.preventDefault();
                    isSwipingReply = true;
                    swipeProgress = Math.min(100, (deltaX / 50) * 100);
                    messageElement.style.transform = \`translateX(\${Math.min(15, deltaX * 0.3)}px)\`;
                    
                    if (swipeProgress >= 80) {
                        showSwipeIndicator(t().swipeReply, 400);
                        if (onReplyCallback) onReplyCallback(messageId);
                        messageElement.style.transform = '';
                        isSwipingReply = false;
                    }
                }
            };
            
            const onTouchEnd = () => {
                if (messageElement) {
                    messageElement.style.transform = '';
                }
                isSwipingReply = false;
            };
            
            messageElement.addEventListener('touchstart', onTouchStart, { passive: true });
            messageElement.addEventListener('touchmove', onTouchMove, { passive: false });
            messageElement.addEventListener('touchend', onTouchEnd);
        }
        
        function setupLongPressForMessage(messageElement, messageId, isOwnMessage, hasText) {
            if (!messageElement) return;
            
            let pressTimer = null;
            let isPressed = false;
            
            const onTouchStart = (e) => {
                isPressed = true;
                pressTimer = setTimeout(() => {
                    if (isPressed) {
                        showMessageActionSheet(messageId, isOwnMessage, hasText);
                        if (navigator.vibrate) navigator.vibrate(50);
                        showSwipeIndicator(t().messageActions, 500);
                    }
                }, 500);
            };
            
            const onTouchEnd = () => {
                isPressed = false;
                if (pressTimer) {
                    clearTimeout(pressTimer);
                    pressTimer = null;
                }
            };
            
            const onTouchMove = () => {
                if (pressTimer) {
                    clearTimeout(pressTimer);
                    pressTimer = null;
                }
                isPressed = false;
            };
            
            messageElement.addEventListener('touchstart', onTouchStart);
            messageElement.addEventListener('touchend', onTouchEnd);
            messageElement.addEventListener('touchmove', onTouchMove);
        }
        
        function showMessageActionSheet(messageId, isOwnMessage, hasText) {
            const overlay = document.createElement('div');
            overlay.style.cssText = 'position:fixed;inset:0;background:rgba(0,0,0,0.5);z-index:10000;display:flex;align-items:flex-end;justify-content:center;';
            
            const sheet = document.createElement('div');
            sheet.style.cssText = 'background:var(--card-bg);border-radius:20px 20px 0 0;width:100%;max-width:500px;padding:20px;animation:slideUp 0.3s ease;';
            sheet.innerHTML = \`
                <div style="text-align:center;padding-bottom:10px;border-bottom:1px solid var(--border-color);margin-bottom:10px;color:var(--text-secondary);font-size:12px;">📋 \${t().messageActions}</div>
                <button onclick="replyToMessageFromAction(\${messageId})" style="margin-bottom:8px;background:var(--message-bg);color:var(--text-primary);">↩️ \${t().swipeReply}</button>
                \${hasText ? \`<button onclick="copyMessageText(\${messageId})" style="margin-bottom:8px;background:var(--message-bg);color:var(--text-primary);">\${t().copyText}</button>\` : ''}
                \${isOwnMessage && hasText ? \`<button onclick="editMessageFromAction(\${messageId})" style="margin-bottom:8px;background:var(--message-bg);color:var(--text-primary);">✏️ \${t().changeDisplayName}</button>\` : ''}
                \${isOwnMessage ? \`<button onclick="deleteMessageFromAction(\${messageId})" style="background:var(--message-bg);color:#f44336;">🗑️ \${t().deleteAccount}</button>\` : ''}
                <button onclick="this.closest('div').parentElement.remove()" style="margin-top:12px;background:var(--border-color);color:var(--text-primary);">✕ \${t().deleteConfirm}</button>
            \`;
            
            overlay.appendChild(sheet);
            document.body.appendChild(overlay);
            
            const style = document.createElement('style');
            style.textContent = \`
                @keyframes slideUp {
                    from { transform: translateY(100%); }
                    to { transform: translateY(0); }
                }
            \`;
            document.head.appendChild(style);
            
            overlay.onclick = (e) => {
                if (e.target === overlay) overlay.remove();
            };
        }
        
        window.replyToMessageFromAction = function(messageId) {
            const msg = allMessages.find(m => m.id === messageId);
            if (msg) {
                replyingTo = msg;
                renderReplyBar();
                document.getElementById('messageInput')?.focus();
            }
            document.querySelector('.profile-modal, [style*="z-index:10000"]')?.remove();
        };
        
        window.copyMessageText = async function(messageId) {
            const msg = allMessages.find(m => m.id === messageId);
            if (msg && msg.text) {
                await navigator.clipboard.writeText(msg.text);
                showSwipeIndicator(t().copied, 1000);
            }
            document.querySelector('.profile-modal, [style*="z-index:10000"]')?.remove();
        };
        
        window.editMessageFromAction = function(messageId) {
            window.startEdit(messageId);
            document.querySelector('.profile-modal, [style*="z-index:10000"]')?.remove();
        };
        
        window.deleteMessageFromAction = function(messageId) {
            if (confirm(t().deleteConfirm)) {
                deleteMessage(messageId);
            }
            document.querySelector('.profile-modal, [style*="z-index:10000"]')?.remove();
        };
        
        function setupEdgeSwipe(container, onEdgeSwipe) {
            if (!container) return;
            
            let touchStartX = 0;
            let touchStartY = 0;
            let isEdgeSwipe = false;
            
            const onTouchStart = (e) => {
                if (e.touches[0].clientX < 20) {
                    touchStartX = e.touches[0].clientX;
                    touchStartY = e.touches[0].clientY;
                    isEdgeSwipe = true;
                }
            };
            
            const onTouchMove = (e) => {
                if (!isEdgeSwipe) return;
                const deltaX = e.touches[0].clientX - touchStartX;
                const deltaY = e.touches[0].clientY - touchStartY;
                
                if (deltaX > 30 && Math.abs(deltaX) > Math.abs(deltaY)) {
                    showSwipeIndicator(t().swipeBack, 300);
                    if (onEdgeSwipe) onEdgeSwipe();
                    isEdgeSwipe = false;
                }
            };
            
            const onTouchEnd = () => {
                isEdgeSwipe = false;
            };
            
            container.addEventListener('touchstart', onTouchStart, { passive: true });
            container.addEventListener('touchmove', onTouchMove, { passive: false });
            container.addEventListener('touchend', onTouchEnd);
        }

        async function refreshOnlineStatuses() {
            if (!currentUser) return;
            try {
                const res = await fetch('/api/users?current=' + currentUser.username);
                const users = await res.json();
                users.forEach(u => {
                    userOnlineStatus[u.username] = isOnline(u.last_seen);
                });
                const feed = document.getElementById('postFeed');
                if (feed && allPosts.length) {
                    renderPostFeed(allPosts);
                }
            } catch(e) {}
        }
        
        function startOnlineStatusUpdates() {
            if (onlineStatusInterval) clearInterval(onlineStatusInterval);
            refreshOnlineStatuses();
            onlineStatusInterval = setInterval(refreshOnlineStatuses, 30000);
        }
        
        window.toggleHamburger = function() {
            const menu = document.getElementById('hamburgerMenu');
            if (!menu) return;
            const isHidden = menu.style.display === 'none' || menu.style.display === '';
            menu.style.display = isHidden ? 'block' : 'none';
        }
        
        window.closeHamburger = function() {
            const menu = document.getElementById('hamburgerMenu');
            if (menu) menu.style.display = 'none';
        }
        
        async function showMain() {
            activeTab = 'login';
            const L = t();
            document.getElementById('app').innerHTML = \`
                <div class="header">
                    <h2>📬 Peykak</h2>
                    <div style="display:flex;gap:6px;">
                        <button class="icon-btn" onclick="toggleDarkMode()">🌙</button>
                        <button class="icon-btn" onclick="showLangPicker()" title="\${L.language}">🌐</button>
                    </div>
                </div>
                <div class="login-box">
                    <div class="tab-buttons">
                        <button class="tab-btn active" onclick="switchTab('login')">\${L.login}</button>
                        <button class="tab-btn" onclick="switchTab('register')">\${L.register}</button>
                    </div>
                    <div id="tabContent"></div>
                </div>
            \`;
            switchTab('login');
        }
        
        window.switchTab = function(tab) {
            activeTab = tab;
            const L = t();
            document.querySelectorAll('.tab-btn').forEach((btn, i) => {
                if ((tab === 'login' && i === 0) || (tab === 'register' && i === 1)) btn.classList.add('active');
                else btn.classList.remove('active');
            });
            const container = document.getElementById('tabContent');
            if (tab === 'login') {
                container.innerHTML = \`
                    <div class="input-group"><label>👤 \${L.username}</label><input type="text" id="username" placeholder="\${L.usernamePlaceholder}"></div>
                    <div class="input-group"><label>🔒 \${L.password}</label><input type="password" id="password" placeholder="\${L.passwordPlaceholder}" onkeypress="if(event.key==='Enter') login()"></div>
                    <button onclick="login()">\${L.loginBtn}</button>
                    <div id="errorMsg" class="error"></div>
                \`;
            } else {
                container.innerHTML = \`
                    <div class="input-group"><label>👤 \${L.username}</label><input type="text" id="regUsername" placeholder="\${L.regUsernamePlaceholder}"></div>
                    <div class="input-group"><label>🔒 \${L.password}</label><input type="password" id="regPassword" placeholder="\${L.regPasswordPlaceholder}"></div>
                    <div class="input-group"><label>📛 \${L.displayName}</label><input type="text" id="regDisplayName" placeholder="\${L.displayNamePlaceholder}"></div>
                    <button onclick="register()">\${L.registerBtn}</button>
                    <div id="regMsg" class="error"></div>
                \`;
            }
        }
        
        window.register = async function() {
            const L = t();
            const username = document.getElementById('regUsername').value.trim();
            const password = document.getElementById('regPassword').value;
            const displayName = document.getElementById('regDisplayName').value.trim() || username;
            if (!username || !password) { document.getElementById('regMsg').innerText = L.fillFields; return; }
            try {
                const res = await fetch('/api/register', { method: 'POST', headers: { 'Content-Type': 'application/json' }, body: JSON.stringify({ username, password, displayName }) });
                const data = await res.json();
                if (data.success) {
                    document.getElementById('regMsg').className = 'success';
                    document.getElementById('regMsg').innerText = L.registerSuccess;
                    setTimeout(() => switchTab('login'), 1500);
                } else { document.getElementById('regMsg').innerText = data.error; }
            } catch(e) { document.getElementById('regMsg').innerText = L.serverError; }
        }
        
        window.login = async function() {
            const L = t();
            const username = document.getElementById('username').value.trim();
            const password = document.getElementById('password').value;
            if (!username || !password) { document.getElementById('errorMsg').innerText = L.fillFields; return; }
            try {
                const res = await fetch('/api/login', { method: 'POST', headers: { 'Content-Type': 'application/json' }, body: JSON.stringify({ username, password }) });
                const data = await res.json();
                if (data.success) {
                    currentUser = data.user;
                    const av = await fetchAvatar(currentUser.username);
                    currentUser.avatar = av;
                    currentUser.bio = bioCache[currentUser.username] || null;
                    avatarCache[currentUser.username] = av;
                    avatarCacheTime[currentUser.username] = Date.now();
                    localStorage.setItem('currentUser', JSON.stringify(currentUser));
                    updateLastSeen();
                    if (lastSeenInterval) clearInterval(lastSeenInterval);
                    lastSeenInterval = setInterval(updateLastSeen, 30000);
                    if (localStorage.getItem('notificationEnabled') === 'true') notificationPermission = true;
                    showUsersList();
                } else { document.getElementById('errorMsg').innerText = data.error; }
            } catch(e) { document.getElementById('errorMsg').innerText = L.serverError; }
        }

        async function refreshUsersList() {
            if (!currentUser) return;
            try {
                const [usersRes] = await Promise.all([
                    fetch('/api/users?current=' + currentUser.username),
                    fetchUnreadCounts()
                ]);
                const users = await usersRes.json();
                const container = document.getElementById('usersList');
                if (!container) return;
                const now = Date.now();
                users.forEach(u => {
                    avatarCache[u.username] = u.avatar || null;
                    avatarCacheTime[u.username] = now;
                    if (u.bio !== undefined) bioCache[u.username] = u.bio || null;
                });

                users.sort((a, b) => {
                    const ua = unreadCounts[a.username] || 0;
                    const ub = unreadCounts[b.username] || 0;
                    return ub - ua;
                });

                const groupCard = \`
                    <div class="user-card group-card" onclick="startGroupChat()">
                        <div class="avatar-placeholder" style="background:linear-gradient(135deg,#667eea,#764ba2);">💬</div>
                        <div>
                            <div class="user-name">\${t().publicGroup}</div>
                            <div class="last-seen-text">\${t().publicGroupSub}</div>
                        </div>
                    </div>
                \`;

                if (users.length === 0) {
                    container.innerHTML = groupCard + \`<div style="text-align:center;color:#999;padding:20px">\${t().noOtherUsers}</div>\`;
                    return;
                }

                container.innerHTML = groupCard + users.map(user => {
                    const unread = unreadCounts[user.username] || 0;
                    const badgeHtml = unread > 0
                        ? \`<div class="unread-badge">\${unread > 99 ? '99+' : unread}</div>\`
                        : '';
                    const hasUnreadClass = unread > 0 ? ' has-unread' : '';
                    const statusInfo = getStatusWithIcon(user.last_seen);
                    const statusHtml = statusInfo.isOnline 
                        ? \`<div class="last-seen-text" style="color: var(--online-green);">🟢 \${t().statusOnline}</div>\`
                        : \`<div class="last-seen-text">⚫ \${statusInfo.text}</div>\`;
                    return \`
                        <div class="user-card\${hasUnreadClass}" onclick="startChat('\${user.username}', '\${escapeHtml(user.display_name)}')">
                            \${avatarHtml(user.avatar || null, user.display_name, 'normal')}
                            <div style="flex:1;overflow:hidden;">
                                <div class="user-name">\${escapeHtml(user.display_name)}\${getUserBadge(user.username)}</div>
                                \${user.bio ? '<div style="font-size:11px;color:var(--text-secondary);margin-top:2px;overflow:hidden;white-space:nowrap;text-overflow:ellipsis;">' + escapeHtml(user.bio) + '</div>' : ''}
                                \${statusHtml}
                            </div>
                            \${badgeHtml}
                        </div>
                    \`;
                }).join('');
                
                const mainContent = document.getElementById('mainTabContent');
                if (mainContent) {
                    setupMainTabSwipes(mainContent);
                }
                
            } catch(e) {}
        }
        
        async function showUsersList(activeMainTab) {
            if (usersRefreshInterval) clearInterval(usersRefreshInterval);
            const tab = activeMainTab || localStorage.getItem('mainTab') || 'messages';
            currentMainTab = tab;
            localStorage.setItem('mainTab', tab);
            const L = t();
            document.getElementById('app').innerHTML = \`
                <div class="header">
                    <span style="font-weight:600;">\${L.greeting(escapeHtml(currentUser.name))}</span>
                    <button class="icon-btn hamburger-btn" onclick="toggleHamburger()">☰</button>
                </div>
                <div id="hamburgerMenu" style="display:none;" class="hamburger-menu">
                    <div class="hamburger-overlay" onclick="closeHamburger()"></div>
                    <div class="hamburger-panel">
                        <div class="hamburger-header">
                            \${avatarHtml(currentUser.avatar || null, currentUser.name, 'normal')}
                            <div>
                                <div class="hamburger-name">\${escapeHtml(currentUser.name)}</div>
                                <div class="hamburger-username">@\${escapeHtml(currentUser.username)}</div>
                            </div>
                        </div>
                        <div class="hamburger-divider"></div>
                        <button class="hamburger-item" onclick="closeHamburger();toggleDarkMode()">
                            <span class="hamburger-icon">🌙</span>
                            <span class="hamburger-label">\${L.darkMode}</span>
                        </button>
                        <button class="hamburger-item" onclick="closeHamburger();enableNotifications()">
                            <span class="hamburger-icon">🔔</span>
                            <span class="hamburger-label">\${L.notifications}</span>
                        </button>
                        <button class="hamburger-item" onclick="closeHamburger();showLangPicker()">
                            <span class="hamburger-icon">🌐</span>
                            <span class="hamburger-label">\${L.language}</span>
                        </button>
                        <button class="hamburger-item" onclick="closeHamburger();showSettings()">
                            <span class="hamburger-icon">⚙️</span>
                            <span class="hamburger-label">\${L.settings}</span>
                        </button>
                        <div class="hamburger-divider"></div>
                        <button class="hamburger-item danger-menu-item" onclick="closeHamburger();logout()">
                            <span class="hamburger-icon">🚪</span>
                            <span class="hamburger-label">\${L.logout}</span>
                        </button>
                    </div>
                </div>
                <div class="main-tabs">
                    <button class="main-tab-btn \${tab === 'messages' ? 'active' : ''}" onclick="showUsersList('messages')">\${L.tabMessages}</button>
                    <button class="main-tab-btn \${tab === 'posts' ? 'active' : ''}" onclick="showUsersList('posts')">\${L.tabTimeline}</button>
                </div>
                <div id="mainTabContent" style="flex:1;display:flex;flex-direction:column;overflow:hidden;"></div>
            \`;
            if (tab === 'messages') {
                document.getElementById('mainTabContent').innerHTML = \`
                    <div class="users-box">
                        <h3 style="margin-bottom: 15px; color: var(--text-primary);">📋 \${L.selectContact}</h3>
                        <div id="usersList"></div>
                    </div>
                \`;
                await refreshUsersList();
                usersRefreshInterval = setInterval(refreshUsersList, 15000);
                const container = document.getElementById('mainTabContent');
                if (container) {
                    setupMainTabSwipes(container);
                }
            } else {
                renderPostsTab();
            }
        }
        
        async function enableNotifications() {
            const L = t();
            const granted = await requestNotificationPermission();
            if (granted) alert(L.notifGranted);
            else alert(L.notifDenied);
        }
        
        window.showSettings = function() {
            const L = t();
            const myAvatar = avatarHtml(currentUser.avatar || null, currentUser.name, 'large');
            const myBio = escapeHtml(currentUser.bio || '');
            const isAdmin = currentUser.username === 'mostafa';
            document.getElementById('app').innerHTML = \`
                <div class="header">
                    <div style="display:flex;align-items:center;gap:10px;">
                        <button class="back-btn" onclick="showUsersList()">←</button>
                        <span>⚙️ \${L.settings}</span>
                    </div>
                    <div style="display:flex;gap:6px;">
                        <button class="icon-btn" onclick="toggleDarkMode()">🌙</button>
                        <button class="icon-btn" onclick="logout()">🚪</button>
                    </div>
                </div>
                <div class="settings-box">
                    <div class="settings-menu">
                        <div class="avatar-upload-area" onclick="document.getElementById('avatarInput').click()">
                            \${myAvatar}
                            <div style="font-size:13px;color:var(--text-secondary);">\${L.changeAvatar}</div>
                            <div style="font-size:11px;color:#667eea;">\${L.changeAvatarHint}</div>
                        </div>
                        <input type="file" id="avatarInput" accept="image/*" style="display:none" onchange="uploadAvatar(this)">
                        <div id="avatarMsg" class="success" style="display:none;margin-top:-10px;margin-bottom:4px;"></div>

                        <div class="settings-item" style="cursor:default;">
                            <div class="settings-item-title" style="margin-bottom:8px;">📝 \${L.bio}</div>
                            <textarea id="bioInput" rows="3" placeholder="\${L.bioPlaceholder}" style="width:100%;padding:10px;border-radius:10px;border:1px solid var(--border-color);background:var(--input-bg);color:var(--text-primary);font-size:13px;resize:none;font-family:inherit;">\${myBio}</textarea>
                            <button onclick="saveBio()" style="margin-top:8px;padding:8px;font-size:13px;border-radius:10px;">\${L.saveBio}</button>
                            <div id="bioMsg" class="success" style="display:none;margin-top:6px;"></div>
                        </div>

                        <div class="settings-item" style="cursor:default;">
                            <div class="settings-item-title" style="margin-bottom:8px;">✏️ \${L.changeDisplayName}</div>
                            <div style="display:flex;gap:8px;align-items:center;">
                                <input type="text" id="displayNameInput" value="\${escapeHtml(currentUser.name)}" placeholder="\${L.displayNamePlaceholder}" style="flex:1;padding:10px;border-radius:10px;border:1px solid var(--border-color);background:var(--input-bg);color:var(--text-primary);font-size:13px;">
                                <button onclick="updateDisplayName()" style="width:auto;padding:10px 16px;border-radius:10px;font-size:13px;">\${L.saveDisplayName}</button>
                            </div>
                            <div id="displayNameMsg" class="success" style="display:none;margin-top:6px;"></div>
                        </div>

                        <div class="settings-item" onclick="enableNotifications()">
                            <div class="settings-item-title">\${L.notifications}</div>
                            <div class="settings-item-desc">\${L.notificationsDesc}</div>
                        </div>

                        <div class="settings-item" onclick="showLangPicker()">
                            <div class="settings-item-title">\${L.language}</div>
                            <div class="settings-item-desc">\${L.languageDesc}</div>
                        </div>

                        \${isAdmin ? \`
                        <div class="settings-item admin-item" onclick="showAdminPanel()">
                            <div class="settings-item-title" style="color:#e53935;">\${L.adminPanel}</div>
                            <div class="settings-item-desc">\${L.adminPanelDesc}</div>
                        </div>
                        \` : ''}

                        <div class="settings-item" onclick="showChangePassword()">
                            <div class="settings-item-title">\${L.changePassword}</div>
                            <div class="settings-item-desc">\${L.changePasswordDesc}</div>
                        </div>
                        <div class="settings-item danger-item" onclick="showDeleteAccount()">
                            <div class="settings-item-title danger-text">\${L.deleteAccount}</div>
                            <div class="settings-item-desc">\${L.deleteAccountDesc}</div>
                        </div>
                    </div>
                </div>
            \`;
        }

        window.updateDisplayName = async function() {
            const L = t();
            const newName = document.getElementById('displayNameInput').value.trim();
            if (!newName) { alert(L.displayNameEmpty); return; }
            try {
                const res = await fetch('/api/update-display-name', { method: 'POST', headers: { 'Content-Type': 'application/json' }, body: JSON.stringify({ username: currentUser.username, displayName: newName }) });
                const data = await res.json();
                if (data.success) {
                    currentUser.name = newName;
                    localStorage.setItem('currentUser', JSON.stringify(currentUser));
                    const msg = document.getElementById('displayNameMsg');
                    if (msg) { msg.style.display = 'block'; msg.textContent = L.displayNameSaved; setTimeout(() => msg.style.display = 'none', 2000); }
                } else { alert(data.error || L.serverError); }
            } catch(e) { alert(t().serverError); }
        }

        window.saveBio = async function() {
            const L = t();
            const bio = document.getElementById('bioInput').value.trim();
            try {
                await fetch('/api/update-bio', { method: 'POST', headers: { 'Content-Type': 'application/json' }, body: JSON.stringify({ username: currentUser.username, bio }) });
                currentUser.bio = bio;
                bioCache[currentUser.username] = bio;
                localStorage.setItem('currentUser', JSON.stringify(currentUser));
                const msg = document.getElementById('bioMsg');
                if (msg) { msg.style.display = 'block'; msg.textContent = L.bioSaved; setTimeout(() => msg.style.display = 'none', 2000); }
            } catch(e) { alert(t().serverError); }
        }

        window.uploadAvatar = async function(input) {
            const L = t();
            const file = input.files[0];
            if (!file) return;
            if (!file.type.startsWith('image/')) { alert(L.onlyImage); return; }
            if (file.size > 2 * 1024 * 1024) { alert(L.avatarTooBig); return; }
            const reader = new FileReader();
            reader.onload = async function(ev) {
                const base64 = ev.target.result;
                try {
                    await fetch('/api/update-avatar', { method: 'POST', headers: { 'Content-Type': 'application/json' }, body: JSON.stringify({ username: currentUser.username, avatar: base64 }) });
                    currentUser.avatar = base64;
                    await refreshAvatar(currentUser.username);
                    avatarCache[currentUser.username] = base64;
                    localStorage.setItem('currentUser', JSON.stringify(currentUser));
                    const area = document.querySelector('.avatar-upload-area');
                    if (area) area.innerHTML = avatarHtml(base64, currentUser.name, 'large') + \`<div style="font-size:13px;color:var(--text-secondary);">\${L.changeAvatar}</div><div style="font-size:11px;color:#667eea;">\${L.changeAvatarHint}</div>\`;
                    const msg = document.getElementById('avatarMsg');
                    if (msg) { msg.style.display = 'block'; msg.textContent = L.avatarSaved; setTimeout(() => msg.style.display = 'none', 2500); }
                } catch(e) { alert(L.serverError); }
            };
            reader.readAsDataURL(file);
            input.value = '';
        }

        window.showAdminPanel = async function() {
            const L = t();
            document.getElementById('app').innerHTML = \`
                <div class="header" style="background:linear-gradient(135deg,#e53935,#b71c1c);">
                    <div style="display:flex;align-items:center;gap:10px;">
                        <button class="back-btn" onclick="showSettings()">←</button>
                        <span style="font-weight:bold;">\${L.adminPanel}</span>
                    </div>
                    <button class="icon-btn" onclick="toggleDarkMode()">🌙</button>
                </div>
                <div class="settings-box">
                    <div class="admin-panel-header">
                        <div class="admin-icon">🛡️</div>
                        <div>
                            <div class="admin-title">\${L.adminTitle}</div>
                            <div class="admin-subtitle">\${L.adminSubtitle}</div>
                        </div>
                    </div>
                    <div id="adminUsersList">
                        <div style="text-align:center;color:#999;padding:30px;">\${L.loading}</div>
                    </div>
                </div>
            \`;
            await loadAdminUsers();
        }

        async function loadAdminUsers() {
            const L = t();
            try {
                const res = await fetch('/api/users?current=__nobody__');
                const users = await res.json();
                const container = document.getElementById('adminUsersList');
                if (!container) return;
                if (users.length === 0) {
                    container.innerHTML = \`<div style="text-align:center;color:#999;padding:20px;">\${L.noOtherUsers}</div>\`;
                    return;
                }
                container.innerHTML = users.map(user => {
                    const statusInfo = getStatusWithIcon(user.last_seen);
                    const statusHtml = statusInfo.isOnline 
                        ? \`<span style="color: var(--online-green);">🟢 \${L.statusOnline}</span>\`
                        : \`<span style="color: var(--offline-gray);">⚫ \${statusInfo.text}</span>\`;
                    return \`
                        <div class="admin-user-row">
                            \${avatarHtml(user.avatar || null, user.display_name, 'normal')}
                            <div class="admin-user-info">
                                <div class="admin-user-name">\${escapeHtml(user.display_name)}\${getUserBadge(user.username)}</div>
                                <div class="admin-user-meta">@\${escapeHtml(user.username)} • \${statusHtml}</div>
                            </div>
                            \${user.username !== 'mostafa'
                                ? \`<button class="admin-delete-btn" onclick="adminDeleteUser('\${user.username}', '\${escapeHtml(user.display_name)}')">\${L.adminDeleteBtn}</button>\`
                                : \`<span class="admin-self-badge">\${L.adminSelfBadge}</span>\`
                            }
                        </div>
                    \`;
                }).join('');
            } catch(e) {
                const container = document.getElementById('adminUsersList');
                if (container) container.innerHTML = \`<div style="text-align:center;color:#f44336;padding:20px;">❌ \${L.serverError}</div>\`;
            }
        }

        window.adminDeleteUser = async function(username, displayName) {
            const L = t();
            if (!confirm(L.adminDeleteConfirm(displayName, username))) return;
            try {
                const res = await fetch('/api/admin-delete-user', {
                    method: 'POST',
                    headers: { 'Content-Type': 'application/json' },
                    body: JSON.stringify({ adminUsername: currentUser.username, targetUsername: username })
                });
                const data = await res.json();
                if (data.success) {
                    showToast('✅', L.adminDeleteSuccess(displayName), null);
                    await loadAdminUsers();
                } else {
                    alert('❌ ' + (data.error || L.serverError));
                }
            } catch(e) { alert('❌ ' + L.serverError); }
        }
        
        window.showChangePassword = function() {
            const L = t();
            document.getElementById('app').innerHTML = \`
                <div class="header">
                    <div style="display:flex;align-items:center;gap:10px;">
                        <button class="back-btn" onclick="showSettings()">←</button>
                        <span>\${L.changePassword}</span>
                    </div>
                    <button class="icon-btn" onclick="toggleDarkMode()">🌙</button>
                </div>
                <div class="login-box">
                    <div class="input-group"><label>🔒 \${L.currentPassword}</label><input type="password" id="oldPassword" placeholder="\${L.currentPasswordPlaceholder}"></div>
                    <div class="input-group"><label>🆕 \${L.newPassword}</label><input type="password" id="newPassword" placeholder="\${L.newPasswordPlaceholder}"></div>
                    <div class="input-group"><label>✅ \${L.confirmPassword}</label><input type="password" id="confirmPassword" placeholder="\${L.confirmPasswordPlaceholder}"></div>
                    <button onclick="changePassword()">\${L.changePasswordBtn}</button>
                    <div id="passMsg" class="error"></div>
                </div>
            \`;
        }
        
        window.changePassword = async function() {
            const L = t();
            const oldPassword = document.getElementById('oldPassword').value;
            const newPassword = document.getElementById('newPassword').value;
            const confirmPassword = document.getElementById('confirmPassword').value;
            if (!oldPassword || !newPassword) { document.getElementById('passMsg').innerText = L.fillFields; return; }
            if (newPassword !== confirmPassword) { document.getElementById('passMsg').innerText = L.passwordMismatch; return; }
            try {
                const res = await fetch('/api/change-password', { method: 'POST', headers: { 'Content-Type': 'application/json' }, body: JSON.stringify({ username: currentUser.username, oldPassword, newPassword }) });
                const data = await res.json();
                if (data.success) {
                    document.getElementById('passMsg').className = 'success';
                    document.getElementById('passMsg').innerText = L.passwordChanged;
                    setTimeout(() => showSettings(), 1500);
                } else { document.getElementById('passMsg').innerText = data.error || L.wrongPassword; }
            } catch(e) { document.getElementById('passMsg').innerText = L.serverError; }
        }
        
        window.showDeleteAccount = function() {
            const L = t();
            document.getElementById('app').innerHTML = \`
                <div class="header">
                    <div style="display:flex;align-items:center;gap:10px;">
                        <button class="back-btn" onclick="showSettings()">←</button>
                        <span>\${L.deleteAccount}</span>
                    </div>
                    <button class="icon-btn" onclick="toggleDarkMode()">🌙</button>
                </div>
                <div class="login-box">
                    <div class="input-group"><label>\${L.deleteWarning}</label><input type="password" id="deletePassword" placeholder="\${L.deletePasswordPlaceholder}"></div>
                    <button onclick="deleteAccount()" style="background:linear-gradient(135deg,#f44336,#d32f2f);">\${L.deleteAccountBtn}</button>
                    <div id="deleteMsg" class="error"></div>
                </div>
            \`;
        }
        
        window.deleteAccount = async function() {
            const L = t();
            const password = document.getElementById('deletePassword').value;
            if (!password) { document.getElementById('deleteMsg').innerText = L.fillFields; return; }
            if (!confirm(L.deleteConfirmAlert)) return;
            try {
                const res = await fetch('/api/delete-account', { method: 'POST', headers: { 'Content-Type': 'application/json' }, body: JSON.stringify({ username: currentUser.username, password }) });
                const data = await res.json();
                if (data.success) { alert(L.deleteAccount); logout(); }
                else { document.getElementById('deleteMsg').innerText = data.error || L.wrongPassword; }
            } catch(e) { document.getElementById('deleteMsg').innerText = L.serverError; }
        }
        
        async function startChat(username, displayName) {
            const oldMessagesDiv = document.getElementById('messages');
            if (oldMessagesDiv && oldMessagesDiv._touchStart) {
                oldMessagesDiv.removeEventListener('touchstart', oldMessagesDiv._touchStart);
                oldMessagesDiv.removeEventListener('touchend', oldMessagesDiv._touchEnd);
            }
            if (usersRefreshInterval) clearInterval(usersRefreshInterval);
            if (refreshInterval) clearInterval(refreshInterval);
            currentChatUser = { username, displayName };
            replyingTo = null;
            pendingImage = null;
            allMessages = [];
            lastMsgId = 0;

            if (unreadCounts[username]) {
                unreadCounts[username] = 0;
            }

            await fetchAvatar(username);
            const partnerBio = bioCache[username] || '';
            const chatAvatar = avatarHtml(avatarCache[username] || null, displayName, 'msg');
            const chatBadge = getUserBadge(username);
            const chatBioHtml = partnerBio ? '<div style="font-size:10px;opacity:0.8;color:rgba(255,255,255,0.85);overflow:hidden;white-space:nowrap;text-overflow:ellipsis;max-width:160px;">' + escapeHtml(partnerBio) + '</div>' : '';

            let userData = null;
            try {
                const userRes = await fetch('/api/users?current=' + currentUser.username);
                const usersList = await userRes.json();
                userData = usersList.find(u => u.username === username);
            } catch(e) {}
            
            let statusDisplay = '';
            if (userData) {
                const statusInfo = getStatusWithIcon(userData.last_seen);
                if (statusInfo.isOnline) {
                    statusDisplay = '<div style="font-size:9px;color:#4caf50;">🟢 ' + t().statusOnline + '</div>';
                } else {
                    statusDisplay = '<div style="font-size:9px;color:rgba(255,255,255,0.7);">⚫ ' + statusInfo.text + '</div>';
                }
            }

            document.getElementById('app').innerHTML = \`
                <div class="header">
                    <div style="display:flex;align-items:center;gap:8px;flex:1;overflow:hidden;cursor:pointer;" onclick="showProfile('\${username}','\${displayName}')">
                        <button class="back-btn" onclick="event.stopPropagation();showUsersList()" style="flex-shrink:0;width:auto;padding:6px 10px;">←</button>
                        \${chatAvatar}
                        <div style="overflow:hidden;line-height:1.3;">
                            <div style="font-weight:700;font-size:15px;white-space:nowrap;">\${escapeHtml(displayName)}\${chatBadge}</div>
                            \${chatBioHtml}
                            \${statusDisplay}
                        </div>
                    </div>
                    <div style="display:flex;gap:6px;flex-shrink:0;">
                        <button class="icon-btn" onclick="toggleDarkMode()">🌙</button>
                        <button class="icon-btn" onclick="showSettings()">⚙️</button>
                        <button class="icon-btn" onclick="logout()">🚪</button>
                    </div>
                </div>
                <div class="messages" id="messages"></div>
                <div id="replyBarContainer"></div>
                <div id="imagePreviewContainer"></div>
                <div class="input-area">
                    <button class="upload-btn" onclick="document.getElementById('fileInput').click()">🖼️</button>
                    <input type="text" id="messageInput" placeholder="\${t().messagePlaceholder}" onkeypress="if(event.key==='Enter') sendMessage()">
                    <button onclick="sendMessage()">📤</button>
                </div>
            \`;
            
            const messagesContainer = document.getElementById('messages');
            if (messagesContainer) {
                setupChatBackSwipe(messagesContainer, () => {
                    showUsersList();
                });
            }
            
            setupEdgeSwipe(document.getElementById('app'), () => {
                showUsersList();
            });
            
            document.getElementById('fileInput').addEventListener('change', handleFileSelect);
            await loadMessages(true);
            refreshInterval = setInterval(() => loadMessages(false), 2000);
        }

        async function startGroupChat() {
            if (refreshInterval) clearInterval(refreshInterval);
            if (usersRefreshInterval) clearInterval(usersRefreshInterval);
            currentChatUser = null;
            replyingTo = null;
            pendingImage = null;
            allMessages = [];
            lastMsgId = 0;

            document.getElementById('app').innerHTML = \`
                <div class="header" style="flex-shrink:0;">
                    <div style="display:flex;align-items:center;gap:10px;">
                        <button class="back-btn" onclick="showUsersList()">←</button>
                        <span style="font-weight:bold;">💬 \${t().publicGroup}</span>
                    </div>
                    <div style="display:flex;gap:6px;">
                        <button class="icon-btn" onclick="toggleDarkMode()">🌙</button>
                        <button class="icon-btn" onclick="showSettings()">⚙️</button>
                        <button class="icon-btn" onclick="logout()">🚪</button>
                    </div>
                </div>
                <div class="messages" id="messages"></div>
                <div id="replyBarContainer"></div>
                <div id="imagePreviewContainer"></div>
                <div class="input-area">
                    <button class="upload-btn" onclick="document.getElementById('fileInput').click()">🖼️</button>
                    <input type="text" id="messageInput" placeholder="\${t().groupMessagePlaceholder}" onkeypress="if(event.key==='Enter') sendGroupMessage()">
                    <button onclick="sendGroupMessage()">📤</button>
                </div>
            \`;

            const messagesContainer = document.getElementById('messages');
            if (messagesContainer) {
                setupChatBackSwipe(messagesContainer, () => {
                    showUsersList();
                });
            }
            
            setupEdgeSwipe(document.getElementById('app'), () => {
                showUsersList();
            });

            document.getElementById('fileInput').addEventListener('change', handleFileSelect);
            await loadGroupMessages(true);
            refreshInterval = setInterval(() => loadGroupMessages(false), 2000);
        }

        async function loadGroupMessages(fullLoad) {
            if (!currentUser) return;
            try {
                const url = fullLoad ? '/api/group-messages' : '/api/group-messages?since=' + lastMsgId;
                const res = await fetch(url);
                const messages = await res.json();

                if (fullLoad) {
                    allMessages = messages;
                    if (messages.length) lastMsgId = messages[messages.length - 1].id;
                    const senders = [...new Set(messages.map(m => m.sender))];
                    await Promise.all(senders.map(s => fetchAvatar(s)));
                    scheduleRender(() => renderGroupMessages(allMessages));
                } else {
                    if (!messages || !messages.length) return;
                    const newMsgs = messages.filter(m => m.id > lastMsgId);
                    if (!newMsgs.length) return;
                    const newSenders = [...new Set(newMsgs.map(m => m.sender).filter(s => !avatarCacheTime[s]))];
                    if (newSenders.length) await Promise.all(newSenders.map(s => fetchAvatar(s)));
                    newMsgs.forEach(m => { if (!allMessages.find(o => o.id === m.id)) allMessages.push(m); });
                    lastMsgId = allMessages[allMessages.length - 1].id;
                    const lastNew = newMsgs[newMsgs.length - 1];
                    if (lastNew.sender !== currentUser.username) {
                        const preview = lastNew.image_data ? '🖼️ تصویر' : (lastNew.text || '').substring(0, 50);
                        sendNotification('💬 گفتگوی عمومی', preview, () => startGroupChat());
                    }
                    scheduleRender(() => renderGroupMessages(allMessages));
                }
            } catch(e) { console.log('خطا:', e); }
        }

        function renderGroupMessages(messages) {
            const container = document.getElementById('messages');
            if (!container) return;
            const L = t();
            const wasScrolledToBottom = container.scrollHeight - container.scrollTop - container.clientHeight < 50;
            if (!messages || messages.length === 0) { container.innerHTML = \`<div style="text-align:center;color:#999;padding:20px">\${L.emptyGroup}</div>\`; return; }
            const msgMap = {};
            messages.forEach(m => msgMap[m.id] = m);
            let lastDate = '', messagesHtml = '';
            const locale = currentLang === 'fa' ? 'fa-IR' : 'en-US';
            messages.forEach(msg => {
                const utcDate = new Date(msg.created_at);
                const tehranTime = new Date(utcDate.getTime() + (3.5 * 60 * 60 * 1000));
                const todayTehran = new Date(new Date().getTime() + (3.5 * 60 * 60 * 1000));
                const yesterday = new Date(todayTehran); yesterday.setDate(yesterday.getDate() - 1);
                const dateStr = tehranTime.toLocaleDateString(locale);
                if (dateStr !== lastDate) {
                    lastDate = dateStr;
                    let dateTitle = dateStr;
                    if (tehranTime.toDateString() === todayTehran.toDateString()) dateTitle = L.today;
                    else if (tehranTime.toDateString() === yesterday.toDateString()) dateTitle = L.yesterday;
                    messagesHtml += '<div class="date-separator"><span>' + dateTitle + '</span></div>';
                }
                const isMe = msg.sender === currentUser.username;
                const hours = tehranTime.getHours().toString().padStart(2, '0');
                const minutes = tehranTime.getMinutes().toString().padStart(2, '0');
                const time = hours + ':' + minutes;
                let replyHtml = '';
                if (msg.reply_to_id && msgMap[msg.reply_to_id]) {
                    const parentMsg = msgMap[msg.reply_to_id];
                    const parentPreview = parentMsg.image_data ? '🖼️' : escapeHtml((parentMsg.text || '').substring(0, 60));
                    replyHtml = '<div class="reply-preview" onclick="scrollToMessage(' + msg.reply_to_id + ')"><div class="reply-sender">' + escapeHtml(parentMsg.sender) + '</div><div class="reply-text">' + parentPreview + '</div></div>';
                }
                let imageHtml = '';
                if (msg.image_data) imageHtml = '<img class="msg-image" src="' + msg.image_data + '" alt="img" loading="lazy" onclick="openImageModal(this.src)">';
                const editedLabel = msg.is_edited ? \`<span class="edited-label">(\${L.edited})</span>\` : '';
                const textHtml = msg.text ? '<div>' + escapeHtml(msg.text) + editedLabel + '</div>' : (editedLabel ? '<div>' + editedLabel + '</div>' : '');
                let actionsHtml = '<div class="message-actions"><button class="action-btn" onclick="startGroupReply(' + msg.id + ')">↩️</button>';
                if (isMe) {
                    if (msg.text) actionsHtml += '<button class="action-btn" onclick="startGroupEdit(' + msg.id + ')">✏️</button>';
                    actionsHtml += '<button class="action-btn" onclick="deleteGroupMessage(' + msg.id + ')">🗑️</button>';
                }
                actionsHtml += '</div>';
                const messageIdAttr = \`id="msg-\${msg.id}"\`;
                messagesHtml += '<div class="message-row ' + (isMe ? 'sent-row' : '') + '">' +
                    avatarHtml(avatarCache[msg.sender] || null, msg.sender, 'msg') +
                    '<div class="message ' + (isMe ? 'sent' : 'received') + '" ' + messageIdAttr + '>' + actionsHtml + '<div class="sender">' + escapeHtml(msg.sender) + (isMe ? '' : getUserBadge(msg.sender)) + '</div>' + replyHtml + '<div class="msg-text">' + textHtml + imageHtml + '</div><div class="time">' + time + '</div></div>' +
                    '</div>';
            });
            container.innerHTML = messagesHtml;
            
            messages.forEach(msg => {
                const msgElement = document.getElementById('msg-' + msg.id);
                if (msgElement) {
                    setupMessageReplySwipe(msgElement, msg.id, (id) => {
                        window.startGroupReply(id);
                    });
                    setupLongPressForMessage(msgElement, msg.id, msg.sender === currentUser.username, !!msg.text);
                }
            });
            
            if (wasScrolledToBottom) container.scrollTop = container.scrollHeight;
        }

        async function sendGroupMessage() {
            const input = document.getElementById('messageInput');
            const text = input ? input.value.trim() : '';
            if (!text && !pendingImage) return;
            try {
                await fetch('/api/group-messages', { method: 'POST', headers: { 'Content-Type': 'application/json' }, body: JSON.stringify({ text: text || "", sender: currentUser.username, image_data: pendingImage || null, reply_to_id: replyingTo ? replyingTo.id : null }) });
                if (input) input.value = '';
                pendingImage = null;
                replyingTo = null;
                renderImagePreview();
                renderReplyBar();
                await loadGroupMessages(true);
            } catch(e) { console.log('خطا در ارسال گروهی:', e); }
        }

        async function deleteGroupMessage(messageId) {
            if (confirm(t().deleteConfirm)) {
                await fetch('/api/delete-group-message', { method: 'POST', headers: { 'Content-Type': 'application/json' }, body: JSON.stringify({ messageId, sender: currentUser.username }) });
                await loadGroupMessages(true);
            }
        }

        window.startGroupReply = function(messageId) {
            const msg = allMessages.find(m => m.id === messageId);
            if (!msg) return;
            replyingTo = msg;
            renderReplyBar();
            document.getElementById('messageInput').focus();
        }

        window.startGroupEdit = function(messageId) {
            const msg = allMessages.find(m => m.id === messageId);
            if (!msg) return;
            const msgEl = document.getElementById('msg-' + messageId);
            if (!msgEl) return;
            const textDiv = msgEl.querySelector('.msg-text');
            if (!textDiv) return;
            textDiv.innerHTML = \`<div class="edit-area"><textarea id="editInput-\${messageId}" rows="2">\${escapeHtml(msg.text)}</textarea><div class="edit-buttons"><button class="edit-save-btn" onclick="saveGroupEdit(\${messageId})">✅ ذخیره</button><button class="edit-cancel-btn" onclick="loadGroupMessages(true)">لغو</button></div></div>\`;
            const ta = document.getElementById('editInput-' + messageId);
            if (ta) { ta.focus(); ta.select(); }
        }

        window.saveGroupEdit = async function(messageId) {
            const ta = document.getElementById('editInput-' + messageId);
            if (!ta) return;
            const newText = ta.value.trim();
            if (!newText) return;
            await fetch('/api/edit-group-message', { method: 'POST', headers: { 'Content-Type': 'application/json' }, body: JSON.stringify({ messageId, sender: currentUser.username, newText }) });
            await loadGroupMessages(true);
        }
        
        function handleFileSelect(e) {
            const file = e.target.files[0];
            if (!file) return;
            if (!file.type.startsWith('image/')) { alert('لطفاً فقط فایل تصویری انتخاب کنید'); return; }
            if (file.size > 3 * 1024 * 1024) { alert('حجم عکس نباید بیشتر از ۳ مگابایت باشد'); e.target.value = ''; return; }
            const reader = new FileReader();
            reader.onload = function(ev) { pendingImage = ev.target.result; renderImagePreview(); };
            reader.readAsDataURL(file);
            e.target.value = '';
        }
        
        function renderImagePreview() {
            const container = document.getElementById('imagePreviewContainer');
            if (!container) return;
            if (!pendingImage) { container.innerHTML = ''; return; }
            container.innerHTML = \`<div class="image-preview-bar"><img src="\${pendingImage}" alt="پیش‌نمایش"><span>عکس آماده ارسال</span><button class="remove-img-btn" onclick="removePendingImage()">✕</button></div>\`;
        }
        
        window.removePendingImage = function() { pendingImage = null; renderImagePreview(); }
        
        window.startReply = function(messageId) {
            const msg = allMessages.find(m => m.id === messageId);
            if (!msg) return;
            replyingTo = msg;
            renderReplyBar();
            document.getElementById('messageInput').focus();
        }
        
        function renderReplyBar() {
            const container = document.getElementById('replyBarContainer');
            if (!container) return;
            if (!replyingTo) { container.innerHTML = ''; return; }
            const senderName = replyingTo.sender === currentUser.username ? currentUser.name : (currentChatUser ? currentChatUser.displayName : replyingTo.sender);
            const previewText = replyingTo.image_data ? '🖼️ تصویر' : escapeHtml(replyingTo.text.substring(0, 60));
            container.innerHTML = \`<div class="reply-bar"><div class="reply-bar-content"><div class="reply-bar-line"></div><div class="reply-bar-text"><div class="reply-bar-sender">\${senderName}</div><div class="reply-bar-msg">\${previewText}</div></div></div><button class="reply-cancel-btn" onclick="cancelReply()">✕</button></div>\`;
        }
        
        window.cancelReply = function() { replyingTo = null; renderReplyBar(); }
        
        window.startEdit = function(messageId) {
            const msg = allMessages.find(m => m.id === messageId);
            if (!msg) return;
            const msgEl = document.getElementById('msg-' + messageId);
            if (!msgEl) return;
            const textDiv = msgEl.querySelector('.msg-text');
            if (!textDiv) return;
            textDiv.innerHTML = \`<div class="edit-area"><textarea id="editInput-\${messageId}" rows="2">\${escapeHtml(msg.text)}</textarea><div class="edit-buttons"><button class="edit-save-btn" onclick="saveEdit(\${messageId})">✅ ذخیره</button><button class="edit-cancel-btn" onclick="cancelEdit(\${messageId})">لغو</button></div></div>\`;
            const ta = document.getElementById('editInput-' + messageId);
            if (ta) { ta.focus(); ta.select(); }
        }
        
        window.saveEdit = async function(messageId) {
            const ta = document.getElementById('editInput-' + messageId);
            if (!ta) return;
            const newText = ta.value.trim();
            if (!newText) return;
            await fetch('/api/edit-message', { method: 'POST', headers: { 'Content-Type': 'application/json' }, body: JSON.stringify({ messageId, sender: currentUser.username, newText }) });
            await loadMessages(true);
        }
        
        window.cancelEdit = async function() { await loadMessages(true); }
        
        function scheduleRender(fn) {
            if (isRendering) { pendingRender = fn; return; }
            isRendering = true;
            requestAnimationFrame(() => {
                fn();
                isRendering = false;
                if (pendingRender) { const p = pendingRender; pendingRender = null; scheduleRender(p); }
            });
        }
        
        async function loadMessages(fullLoad) {
            if (!currentUser || !currentChatUser) return;
            try {
                const base = '/api/messages?user1=' + currentUser.username + '&user2=' + currentChatUser.username;
                const url = fullLoad ? base : base + '&since=' + lastMsgId;
                const res = await fetch(url);
                const messages = await res.json();

                if (fullLoad) {
                    allMessages = messages;
                    if (messages.length) lastMsgId = messages[messages.length - 1].id;
                    const uniqueSenders = [...new Set(messages.map(m => m.sender))];
                    await Promise.all(uniqueSenders.map(s => fetchAvatar(s)));
                    scheduleRender(() => renderMessages(allMessages));
                } else {
                    if (!messages || !messages.length) return;
                    messages.forEach(m => {
                        const idx = allMessages.findIndex(o => o.id === m.id);
                        if (idx >= 0) allMessages[idx] = m;
                        else allMessages.push(m);
                    });
                    if (allMessages.length) lastMsgId = Math.max(...allMessages.map(m => m.id));
                    const newMsgs = messages.filter(m => m.sender === currentChatUser.username);
                    if (newMsgs.length > 0) {
                        const lastMsg = newMsgs[newMsgs.length - 1];
                        const preview = lastMsg.image_data ? '🖼️' : (lastMsg.text || '').substring(0, 50);
                        sendNotification(currentChatUser.displayName, preview, null);
                    }
                    scheduleRender(() => renderMessages(allMessages));
                }

                if (!document.hidden && currentChatUser) {
                    fetch('/api/mark-read', { method: 'POST', headers: { 'Content-Type': 'application/json' }, body: JSON.stringify({ sender: currentChatUser.username, receiver: currentUser.username }) });
                    if (unreadCounts[currentChatUser.username]) {
                        unreadCounts[currentChatUser.username] = 0;
                    }
                }
            } catch(e) { console.log('خطا:', e); }
        }
        
        function renderMessages(messages) {
            const container = document.getElementById('messages');
            if (!container) return;
            const L = t();
            const locale = currentLang === 'fa' ? 'fa-IR' : 'en-US';
            const wasScrolledToBottom = container.scrollHeight - container.scrollTop - container.clientHeight < 50;
            if (!messages || messages.length === 0) { container.innerHTML = \`<div style="text-align:center;color:#999;padding:20px">\${L.emptyChat}</div>\`; return; }
            const msgMap = {};
            messages.forEach(m => msgMap[m.id] = m);
            let lastDate = '', messagesHtml = '';
            messages.forEach(msg => {
                const utcDate = new Date(msg.created_at);
                const tehranTime = new Date(utcDate.getTime() + (3.5 * 60 * 60 * 1000));
                const todayTehran = new Date(new Date().getTime() + (3.5 * 60 * 60 * 1000));
                const yesterday = new Date(todayTehran); yesterday.setDate(yesterday.getDate() - 1);
                const dateStr = tehranTime.toLocaleDateString(locale);
                if (dateStr !== lastDate) {
                    lastDate = dateStr;
                    let dateTitle = dateStr;
                    if (tehranTime.toDateString() === todayTehran.toDateString()) dateTitle = L.today;
                    else if (tehranTime.toDateString() === yesterday.toDateString()) dateTitle = L.yesterday;
                    messagesHtml += '<div class="date-separator"><span>' + dateTitle + '</span></div>';
                }
                const isMe = msg.sender === currentUser.username;
                const senderName = isMe ? currentUser.name : currentChatUser.displayName;
                const hours = tehranTime.getHours().toString().padStart(2, '0');
                const minutes = tehranTime.getMinutes().toString().padStart(2, '0');
                const time = hours + ':' + minutes;
                let readStatus = '';
                if (isMe) readStatus = msg.is_read ? ' ✓✓' : ' ✓';
                let replyHtml = '';
                if (msg.reply_to_id && msgMap[msg.reply_to_id]) {
                    const parentMsg = msgMap[msg.reply_to_id];
                    const parentSenderName = parentMsg.sender === currentUser.username ? currentUser.name : currentChatUser.displayName;
                    const parentPreview = parentMsg.image_data ? '🖼️' : escapeHtml((parentMsg.text || '').substring(0, 60));
                    replyHtml = '<div class="reply-preview" onclick="scrollToMessage(' + msg.reply_to_id + ')"><div class="reply-sender">' + parentSenderName + '</div><div class="reply-text">' + parentPreview + '</div></div>';
                }
                let imageHtml = '';
                if (msg.image_data) imageHtml = '<img class="msg-image" src="' + msg.image_data + '" alt="img" loading="lazy" onclick="openImageModal(this.src)">';
                const editedLabel = msg.is_edited ? \`<span class="edited-label">(\${L.edited})</span>\` : '';
                const textHtml = msg.text ? '<div>' + escapeHtml(msg.text) + editedLabel + '</div>' : (editedLabel ? '<div>' + editedLabel + '</div>' : '');
                let actionsHtml = '<div class="message-actions"><button class="action-btn" onclick="startReply(' + msg.id + ')">↩️</button>';
                if (isMe) {
                    if (msg.text) actionsHtml += '<button class="action-btn" onclick="startEdit(' + msg.id + ')">✏️</button>';
                    actionsHtml += '<button class="action-btn" onclick="deleteMessage(' + msg.id + ')">🗑️</button>';
                }
                actionsHtml += '</div>';
                const messageIdAttr = \`id="msg-\${msg.id}"\`;
                messagesHtml += '<div class="message-row ' + (isMe ? 'sent-row' : '') + '">' +
                    avatarHtml(avatarCache[msg.sender] || null, senderName, 'msg') +
                    '<div class="message ' + (isMe ? 'sent' : 'received') + '" ' + messageIdAttr + '>' + actionsHtml + '<div class="sender">' + senderName + (isMe ? '' : getUserBadge(msg.sender)) + '</div>' + replyHtml + '<div class="msg-text">' + textHtml + imageHtml + '</div><div class="time">' + time + '<span class="read-status">' + readStatus + '</span></div></div>' +
                    '</div>';
            });
            container.innerHTML = messagesHtml;
            
            messages.forEach(msg => {
                const msgElement = document.getElementById('msg-' + msg.id);
                if (msgElement) {
                    setupMessageReplySwipe(msgElement, msg.id, (id) => {
                        window.startReply(id);
                    });
                    setupLongPressForMessage(msgElement, msg.id, msg.sender === currentUser.username, !!msg.text);
                }
            });
            
            if (wasScrolledToBottom) container.scrollTop = container.scrollHeight;
        }
        
        window.scrollToMessage = function(messageId) {
            const el = document.getElementById('msg-' + messageId);
            if (el) { el.scrollIntoView({ behavior: 'smooth', block: 'center' }); el.style.outline = '2px solid #667eea'; setTimeout(() => { el.style.outline = ''; }, 1500); }
        }
        
        async function sendMessage() {
            const input = document.getElementById('messageInput');
            const text = input ? input.value.trim() : '';
            if (!text && !pendingImage) return;
            if (!currentChatUser) return;
            try {
                await fetch('/api/messages', { method: 'POST', headers: { 'Content-Type': 'application/json' }, body: JSON.stringify({ text: text || "", sender: currentUser.username, receiver: currentChatUser.username, image_data: pendingImage || null, reply_to_id: replyingTo ? replyingTo.id : null }) });
                if (input) input.value = '';
                pendingImage = null;
                replyingTo = null;
                renderImagePreview();
                renderReplyBar();
                await loadMessages(true);
            } catch(e) { console.log('خطا در ارسال:', e); }
        }
        
        async function deleteMessage(messageId) {
            if (confirm(t().deleteConfirm)) {
                await fetch('/api/delete-message', { method: 'POST', headers: { 'Content-Type': 'application/json' }, body: JSON.stringify({ messageId, sender: currentUser.username }) });
                await loadMessages(true);
            }
        }
        
        window.openImageModal = function(src) {
            const modal = document.createElement('div');
            modal.className = 'img-modal';
            modal.innerHTML = '<img src="' + src + '" alt="تصویر">';
            modal.onclick = () => document.body.removeChild(modal);
            document.body.appendChild(modal);
        }

        // ==================== سیستم پست‌ها ====================

        function renderPostsTab() {
            if (postRefreshInterval) clearInterval(postRefreshInterval);
            const container = document.getElementById('mainTabContent');
            if (!container) return;
            container.innerHTML = \`<div id="postTabBody" style="flex:1;display:flex;flex-direction:column;overflow:hidden;"></div>\`;
            renderPublicFeed();
            setupMainTabSwipes(container);
        }

        function renderPublicFeed() {
            const L = t();
            const body = document.getElementById('postTabBody');
            if (!body) return;
            body.innerHTML = \`
                <div id="composeSection" style="flex-shrink:0;">
                    <div class="compose-box">
                        \${avatarHtml(currentUser.avatar || null, currentUser.name, 'normal')}
                        <textarea id="postTextarea" placeholder="\${L.whatsOnYourMind}" oninput="updateComposeBtn()" rows="2"></textarea>
                    </div>
                    <div id="postImgPreviewContainer"></div>
                    <div class="compose-actions">
                        <div style="display:flex;gap:4px;">
                            <button class="compose-icon-btn" onclick="document.getElementById('postFileInput').click()" title="🖼️">🖼️</button>
                            <input type="file" id="postFileInput" accept="image/*" style="display:none" onchange="handlePostImageSelect(this)">
                        </div>
                        <div style="display:flex;align-items:center;gap:10px;">
                            <span id="charCount" style="font-size:11px;color:var(--text-secondary);"></span>
                            <button class="compose-submit-btn" id="postSubmitBtn" onclick="submitPost()" disabled>\${L.post}</button>
                        </div>
                    </div>
                </div>
                <div class="post-feed" id="postFeed">
                    <div style="text-align:center;padding:30px;color:var(--text-secondary);">\${L.loading}</div>
                </div>
            \`;
            loadPosts(true);
            postRefreshInterval = setInterval(() => loadPosts(false), 8000);
        }

        window.updateComposeBtn = function() {
            const L = t();
            const ta = document.getElementById('postTextarea');
            const btn = document.getElementById('postSubmitBtn');
            const counter = document.getElementById('charCount');
            if (!ta || !btn) return;
            const len = ta.value.length;
            if (counter) counter.textContent = len > 0 ? len + '/280' : '';
            if (len > 280) { ta.value = ta.value.substring(0, 280); }
            btn.disabled = (ta.value.trim().length === 0 && !pendingPostImage);
            if (btn && !btn.disabled) btn.textContent = L.post;
        }

        window.handlePostImageSelect = function(input) {
            const L = t();
            const file = input.files[0];
            if (!file) return;
            if (!file.type.startsWith('image/')) { alert(L.onlyImage); return; }
            if (file.size > 3 * 1024 * 1024) { alert(L.avatarTooBig); input.value = ''; return; }
            const reader = new FileReader();
            reader.onload = ev => {
                pendingPostImage = ev.target.result;
                const c = document.getElementById('postImgPreviewContainer');
                if (c) c.innerHTML = \`<div class="compose-img-preview"><img src="\${pendingPostImage}"><button class="compose-img-remove" onclick="removePendingPostImage()">✕</button></div>\`;
                document.getElementById('postSubmitBtn').disabled = false;
            };
            reader.readAsDataURL(file);
            input.value = '';
        }

        window.removePendingPostImage = function() {
            pendingPostImage = null;
            const c = document.getElementById('postImgPreviewContainer');
            if (c) c.innerHTML = '';
            updateComposeBtn();
        }

        async function submitPost() {
            const ta = document.getElementById('postTextarea');
            const text = ta ? ta.value.trim() : '';
            if (!text && !pendingPostImage) return;
            const btn = document.getElementById('postSubmitBtn');
            if (btn) btn.disabled = true;
            try {
                await fetch('/api/posts', {
                    method: 'POST',
                    headers: { 'Content-Type': 'application/json' },
                    body: JSON.stringify({ text, sender: currentUser.username, image_data: pendingPostImage || null })
                });
                if (ta) ta.value = '';
                pendingPostImage = null;
                const c = document.getElementById('postImgPreviewContainer');
                if (c) c.innerHTML = '';
                updateComposeBtn();
                await loadPosts(true);
            } catch(e) { alert('خطا در ارسال پست'); }
        }

        async function loadPosts(fullLoad) {
            if (!currentUser) return;
            try {
                const res = await fetch('/api/posts?viewer=' + encodeURIComponent(currentUser.username));
                const posts = await res.json();
                if (!posts || !Array.isArray(posts)) return;

                if (fullLoad) {
                    posts.forEach(p => {
                        localLikes[p.id] = !!p.liked_by_me;
                        localReposts[p.id] = !!p.reposted_by_me;
                    });
                    allPosts = posts;
                    const senders = [...new Set(posts.map(p => p.sender))];
                    await Promise.all(senders.map(s => fetchAvatar(s)));
                    renderPostFeed(allPosts);
                    await registerViewsAndUpdate(posts);
                } else {
                    const existingIds = new Set(allPosts.map(p => p.id));
                    const newPosts = posts.filter(p => !existingIds.has(p.id));

                    if (newPosts.length > 0) {
                        newPosts.forEach(p => {
                            localLikes[p.id] = !!p.liked_by_me;
                            localReposts[p.id] = !!p.reposted_by_me;
                        });
                        const newSenders = [...new Set(newPosts.map(p => p.sender).filter(s => !avatarCacheTime[s]))];
                        if (newSenders.length) await Promise.all(newSenders.map(s => fetchAvatar(s)));
                        allPosts = posts;
                        renderPostFeed(allPosts);
                        await registerViewsAndUpdate(newPosts);
                    } else {
                        posts.forEach(serverPost => {
                            const local = allPosts.find(p => p.id === serverPost.id);
                            if (local) {
                                local.likes_count = serverPost.likes_count;
                                local.reposts_count = serverPost.reposts_count;
                                local.views_count = serverPost.views_count;
                                local.comments_count = serverPost.comments_count;
                            }
                            updateSinglePostCounters(serverPost);
                        });
                    }
                }
            } catch(e) {}
        }

        async function registerViewsAndUpdate(posts) {
            await Promise.all(posts.map(async p => {
                try {
                    const res = await fetch('/api/post-view', {
                        method: 'POST',
                        headers: { 'Content-Type': 'application/json' },
                        body: JSON.stringify({ postId: p.id, username: currentUser.username })
                    });
                    const data = await res.json();
                    if (data.newView) {
                        const post = allPosts.find(x => x.id === p.id);
                        if (post) {
                            post.views_count = (post.views_count || 0) + 1;
                            const el = document.getElementById('view-count-' + post.id);
                            if (el) el.textContent = formatCount(post.views_count);
                        }
                    }
                } catch(e) {}
            }));
        }

        function updateSinglePostCounters(post) {
            const likeCount = document.getElementById('like-count-' + post.id);
            if (likeCount) likeCount.textContent = formatCount(post.likes_count);
            const repostCount = document.getElementById('repost-count-' + post.id);
            if (repostCount) repostCount.textContent = formatCount(post.reposts_count);
            const viewCount = document.getElementById('view-count-' + post.id);
            if (viewCount) viewCount.textContent = formatCount(post.views_count);
            const commentCount = document.getElementById('comment-count-' + post.id);
            if (commentCount) commentCount.textContent = formatCount(post.comments_count);
        }

        function parseUTC(str) {
            if (!str) return null;
            const s = str.endsWith('Z') || str.includes('+') ? str : str.replace(' ', 'T') + 'Z';
            return new Date(s);
        }

        function timeAgo(utcDateString) {
            const L = t();
            if (!utcDateString) return '';
            const postDate = parseUTC(utcDateString);
            if (!postDate || isNaN(postDate)) return '';
            const now = new Date();
            const diffSec = Math.floor((now - postDate) / 1000);
            if (diffSec < 0) return L.justNow;
            if (diffSec < 60) return L.justNow;
            if (diffSec < 3600) return L.minutesAgo(Math.floor(diffSec / 60));
            if (diffSec < 86400) return L.hoursAgo(Math.floor(diffSec / 3600));
            if (diffSec < 604800) return L.daysAgo(Math.floor(diffSec / 86400));
            return currentLang === 'fa'
                ? postDate.toLocaleDateString('fa-IR')
                : postDate.toLocaleDateString('en-US', { month: 'short', day: 'numeric' });
        }

        function formatCount(n) {
            if (!n) return '0';
            if (n >= 1000) return (n / 1000).toFixed(1).replace('.0','') + 'k';
            return String(n);
        }

        function renderPostFeed(posts) {
            const feed = document.getElementById('postFeed');
            if (!feed) return;
            if (!posts || posts.length === 0) {
                feed.innerHTML = \`<div class="empty-feed">\${t().emptyFeed}</div>\`;
                return;
            }
            feed.innerHTML = posts.map(post => renderPostCard(post)).join('');
            if (openCommentPostId) {
                const sec = document.getElementById('comments-' + openCommentPostId);
                if (sec) { sec.style.display = 'block'; loadAndRenderComments(openCommentPostId); }
            }
        }

        function renderPostCard(post) {
            const isMe = post.sender === currentUser.username;
            const timeStr = timeAgo(post.created_at);
            const imgHtml = post.image_data
                ? \`<img class="post-image" src="\${post.image_data}" alt="img" loading="lazy" onclick="openImageModal(this.src)">\`
                : '';
            const likedClass = localLikes[post.id] ? 'liked' : '';
            const repostedClass = localReposts[post.id] ? 'reposted' : '';
            const deleteBtn = isMe
                ? \`<button class="post-delete-btn" onclick="deletePost(\${post.id})" title="🗑️">🗑️</button>\`
                : '';
            let statusDot = '';
            if (userOnlineStatus[post.sender] !== undefined) {
                statusDot = userOnlineStatus[post.sender] 
                    ? '<span class="status-dot online"></span>' 
                    : '<span class="status-dot offline"></span>';
            }
            return \`
                <div class="post-card" id="post-\${post.id}">
                    <div class="post-header">
                        \${avatarHtml(avatarCache[post.sender] || null, post.sender, 'normal')}
                        <div class="post-user-info">
                            <div class="post-display-name">\${escapeHtml(post.sender)}\${getUserBadge(post.sender)} \${statusDot}</div>
                            <div class="post-username-time">@\${escapeHtml(post.sender)} · \${timeStr}</div>
                        </div>
                        \${deleteBtn}
                    </div>
                    \${post.text ? \`<div class="post-text">\${escapeHtml(post.text)}</div>\` : ''}
                    \${imgHtml}
                    <div class="post-actions">
                        <button class="post-action-btn" onclick="toggleComments(\${post.id})">
                            <span class="icon">💬</span><span id="comment-count-\${post.id}">\${formatCount(post.comments_count)}</span>
                        </button>
                        <button class="post-action-btn \${repostedClass}" id="repost-btn-\${post.id}" onclick="toggleRepost(\${post.id})">
                            <span class="icon" id="repost-icon-\${post.id}">🔁</span><span id="repost-count-\${post.id}">\${formatCount(post.reposts_count)}</span>
                        </button>
                        <button class="post-action-btn \${likedClass}" id="like-btn-\${post.id}" onclick="toggleLike(\${post.id})">
                            <span class="icon" id="like-icon-\${post.id}">\${localLikes[post.id] ? '❤️' : '🤍'}</span><span id="like-count-\${post.id}">\${formatCount(post.likes_count)}</span>
                        </button>
                        <button class="post-action-btn" style="cursor:default;">
                            <span class="icon">👁️</span><span id="view-count-\${post.id}">\${formatCount(post.views_count)}</span>
                        </button>
                    </div>
                    <div class="comments-section" id="comments-\${post.id}" style="display:none;"></div>
                </div>
            \`;
        }

        window.toggleLike = async function(postId) {
            const btn = document.getElementById('like-btn-' + postId);
            const iconEl = document.getElementById('like-icon-' + postId);
            const countEl = document.getElementById('like-count-' + postId);
            const post = allPosts.find(p => p.id === postId);
            if (!post) return;
            const newLiked = !localLikes[postId];
            localLikes[postId] = newLiked;
            post.likes_count = Math.max(0, (post.likes_count || 0) + (newLiked ? 1 : -1));
            if (btn) btn.className = 'post-action-btn' + (newLiked ? ' liked' : '');
            if (iconEl) iconEl.textContent = newLiked ? '❤️' : '🤍';
            if (countEl) countEl.textContent = formatCount(post.likes_count);
            try {
                await fetch('/api/post-like', {
                    method: 'POST',
                    headers: { 'Content-Type': 'application/json' },
                    body: JSON.stringify({ postId, username: currentUser.username })
                });
            } catch(e) {
                localLikes[postId] = !newLiked;
                post.likes_count = Math.max(0, post.likes_count + (newLiked ? -1 : 1));
                if (btn) btn.className = 'post-action-btn' + (!newLiked ? ' liked' : '');
                if (iconEl) iconEl.textContent = !newLiked ? '❤️' : '🤍';
                if (countEl) countEl.textContent = formatCount(post.likes_count);
            }
        }

        window.toggleRepost = async function(postId) {
            const btn = document.getElementById('repost-btn-' + postId);
            const countEl = document.getElementById('repost-count-' + postId);
            const post = allPosts.find(p => p.id === postId);
            if (!post) return;
            const newReposted = !localReposts[postId];
            localReposts[postId] = newReposted;
            post.reposts_count = Math.max(0, (post.reposts_count || 0) + (newReposted ? 1 : -1));
            if (btn) btn.className = 'post-action-btn' + (newReposted ? ' reposted' : '');
            if (countEl) countEl.textContent = formatCount(post.reposts_count);
            try {
                await fetch('/api/post-repost', {
                    method: 'POST',
                    headers: { 'Content-Type': 'application/json' },
                    body: JSON.stringify({ postId, username: currentUser.username })
                });
            } catch(e) {
                localReposts[postId] = !newReposted;
                post.reposts_count = Math.max(0, post.reposts_count + (newReposted ? -1 : 1));
                if (btn) btn.className = 'post-action-btn' + (!newReposted ? ' reposted' : '');
                if (countEl) countEl.textContent = formatCount(post.reposts_count);
            }
        }

        window.deletePost = async function(postId) {
            if (!confirm(t().deletePostConfirm)) return;
            try {
                await fetch('/api/delete-post', {
                    method: 'POST',
                    headers: { 'Content-Type': 'application/json' },
                    body: JSON.stringify({ postId, sender: currentUser.username })
                });
                await loadPosts(true);
            } catch(e) {}
        }

        window.toggleComments = async function(postId) {
            const sec = document.getElementById('comments-' + postId);
            if (!sec) return;
            if (sec.style.display === 'none' || sec.style.display === '') {
                sec.style.display = 'block';
                openCommentPostId = postId;
                await loadAndRenderComments(postId);
            } else {
                sec.style.display = 'none';
                openCommentPostId = null;
            }
        }

        async function loadAndRenderComments(postId) {
            const L = t();
            const sec = document.getElementById('comments-' + postId);
            if (!sec) return;
            try {
                const res = await fetch('/api/post-comments?post_id=' + postId);
                const comments = await res.json();
                const senders = [...new Set(comments.map(c => c.sender))];
                await Promise.all(senders.map(s => fetchAvatar(s)));
                let html = '';
                if (comments.length === 0) {
                    html = \`<div style="text-align:center;padding:12px;font-size:12px;color:var(--text-secondary);">\${L.noComments}</div>\`;
                } else {
                    html = comments.map(c => {
                        const statusDot = userOnlineStatus[c.sender] !== undefined
                            ? (userOnlineStatus[c.sender] ? '<span class="status-dot online" style="width:8px;height:8px;display:inline-block;margin-right:5px;"></span>' : '')
                            : '';
                        return \`
                        <div class="comment-item">
                            \${avatarHtml(avatarCache[c.sender] || null, c.sender, 'msg')}
                            <div class="comment-body">
                                <div class="comment-sender">\${escapeHtml(c.sender)}\${getUserBadge(c.sender)} \${statusDot}</div>
                                <div class="comment-text">\${escapeHtml(c.text)}</div>
                                <div class="comment-time">\${timeAgo(c.created_at)}</div>
                            </div>
                            \${c.sender === currentUser.username ? \`<button class="comment-del-btn" onclick="deleteComment(\${c.id}, \${postId})">🗑️</button>\` : ''}
                        </div>
                    \`}).join('');
                }
                html += \`
                    <div class="comment-input-row">
                        \${avatarHtml(currentUser.avatar || null, currentUser.name, 'msg')}
                        <input type="text" id="comment-input-\${postId}" placeholder="\${L.commentPlaceholder}" onkeypress="if(event.key==='Enter') submitComment(\${postId})">
                        <button class="comment-send-btn" onclick="submitComment(\${postId})">\${L.sendComment}</button>
                    </div>
                \`;
                sec.innerHTML = html;
            } catch(e) {}
        }

        window.submitComment = async function(postId) {
            const L = t();
            const input = document.getElementById('comment-input-' + postId);
            if (!input) return;
            const text = input.value.trim();
            if (!text) return;
            input.value = '';
            try {
                await fetch('/api/post-comments', {
                    method: 'POST',
                    headers: { 'Content-Type': 'application/json' },
                    body: JSON.stringify({ postId, sender: currentUser.username, text })
                });
                const post = allPosts.find(p => p.id === postId);
                if (post) post.comments_count = (post.comments_count || 0) + 1;
                await loadAndRenderComments(postId);
                const card = document.getElementById('post-' + postId);
                if (card) {
                    const commentBtn = card.querySelector('.post-action-btn');
                    if (commentBtn) commentBtn.querySelector('span:last-child').textContent = formatCount(post ? post.comments_count : 0);
                }
            } catch(e) { alert(L.serverError); }
        }

        window.deleteComment = async function(commentId, postId) {
            if (!confirm(t().deleteCommentConfirm)) return;
            try {
                await fetch('/api/delete-post-comment', {
                    method: 'POST',
                    headers: { 'Content-Type': 'application/json' },
                    body: JSON.stringify({ commentId, sender: currentUser.username })
                });
                const post = allPosts.find(p => p.id === postId);
                if (post) post.comments_count = Math.max(0, (post.comments_count || 1) - 1);
                await loadAndRenderComments(postId);
            } catch(e) {}
        }
        
        window.showLangPicker = function() {
            const L = t();
            const existing = document.getElementById('lang-modal');
            if (existing) { existing.remove(); return; }
            const modal = document.createElement('div');
            modal.id = 'lang-modal';
            modal.style.cssText = 'position:fixed;inset:0;background:rgba(0,0,0,0.5);display:flex;align-items:center;justify-content:center;z-index:2000;padding:20px;';
            modal.innerHTML = \`
                <div style="background:var(--card-bg);border-radius:20px;padding:24px;width:100%;max-width:300px;box-shadow:0 20px 60px rgba(0,0,0,0.3);" onclick="event.stopPropagation()">
                    <div style="font-size:16px;font-weight:700;color:var(--text-primary);margin-bottom:16px;text-align:center;">\${L.selectLanguage}</div>
                    <div style="display:flex;flex-direction:column;gap:10px;">
                        <button onclick="applyLang('fa')" style="padding:14px;border-radius:14px;font-size:15px;background:\${currentLang==='fa'?'linear-gradient(135deg,#667eea,#764ba2)':'var(--message-bg)'};color:\${currentLang==='fa'?'white':'var(--text-primary)'};border:\${currentLang==='fa'?'none':'1px solid var(--border-color)'};">
                            🇮🇷 فارسی
                        </button>
                        <button onclick="applyLang('en')" style="padding:14px;border-radius:14px;font-size:15px;background:\${currentLang==='en'?'linear-gradient(135deg,#667eea,#764ba2)':'var(--message-bg)'};color:\${currentLang==='en'?'white':'var(--text-primary)'};border:\${currentLang==='en'?'none':'1px solid var(--border-color)'};">
                            🇺🇸 English
                        </button>
                    </div>
                </div>
            \`;
            modal.onclick = () => modal.remove();
            document.body.appendChild(modal);
        }

        window.applyLang = function(lang) {
            setLang(lang);
            const modal = document.getElementById('lang-modal');
            if (modal) modal.remove();
            if (currentUser) showUsersList(localStorage.getItem('mainTab') || 'messages');
            else showMain();
        }

        function logout() {
            if (refreshInterval) clearInterval(refreshInterval);
            if (lastSeenInterval) clearInterval(lastSeenInterval);
            if (usersRefreshInterval) clearInterval(usersRefreshInterval);
            if (postRefreshInterval) clearInterval(postRefreshInterval);
            if (onlineStatusInterval) clearInterval(onlineStatusInterval);
            localStorage.removeItem('currentUser');
            localStorage.removeItem('mainTab');
            currentUser = null;
            currentChatUser = null;
            replyingTo = null;
            pendingImage = null;
            allMessages = [];
            lastMsgId = 0;
            unreadCounts = {};
            allPosts = [];
            pendingPostImage = null;
            openCommentPostId = null;
            userOnlineStatus = {};
            showMain();
        }
        
        window.showProfile = async function(username, displayName) {
            await fetchAvatar(username);
            const avatar = avatarCache[username] || null;
            const bio = bioCache[username] || null;
            const avatarBig = avatar
                ? '<img class="profile-big-avatar" src="' + avatar + '" alt="avatar" onclick="openImageModal(this.src)">'
                : '<div class="profile-big-placeholder">' + (displayName || username)[0].toUpperCase() + '</div>';
            let statusReal = 'در حال دریافت...';
            let isOnlineStatus = false;
            try {
                const res = await fetch('/api/users?current=__nobody__');
                const users = await res.json();
                const found = users.find(u => u.username === username);
                if (found) {
                    isOnlineStatus = isOnline(found.last_seen);
                    const statusInfo = getStatusWithIcon(found.last_seen);
                    statusReal = statusInfo.isOnline ? statusInfo.text : statusInfo.text;
                }
            } catch(e) { statusReal = 'آخرین بازدید: نامشخص'; }
            const statusDotHtml = isOnlineStatus 
                ? '<span class="status-dot online"></span>' 
                : '<span class="status-dot offline"></span>';
            const modal = document.createElement('div');
            modal.className = 'profile-modal';
            modal.id = 'profileModal';
            modal.innerHTML = \`
                <div class="profile-card" onclick="event.stopPropagation()">
                    <button class="close-btn" onclick="document.getElementById('profileModal').remove()">✕</button>
                    \${avatarBig}
                    <div class="profile-name">\${escapeHtml(displayName)}\${getUserBadge(username)}</div>
                    <div class="profile-username">@\${escapeHtml(username)}</div>
                    <div class="profile-status">
                        \${statusDotHtml}
                        \${statusReal}
                    </div>
                    \${bio ? '<div class="profile-bio">' + escapeHtml(bio) + '</div>' : ''}
                </div>
            \`;
            modal.onclick = () => modal.remove();
            document.body.appendChild(modal);
        }

        function escapeHtml(text) {
            if (!text) return '';
            const div = document.createElement('div');
            div.textContent = text;
            return div.innerHTML;
        }

        function getUserBadge(username) {
            if (username === 'mostafa') {
                return ' <span style="color:#1d9bf0;font-size:11px;">✓</span><span style="font-size:9px;color:#888;font-weight:400;"> maker</span>';
            }
            return '';
        }

        async function fetchAvatar(username) {
            const now = Date.now();
            if (avatarCacheTime[username] && now - avatarCacheTime[username] < 180000) return avatarCache[username];
            try {
                const res = await fetch('/api/get-avatar?username=' + encodeURIComponent(username));
                const data = await res.json();
                avatarCache[username] = data.avatar || null;
                bioCache[username] = data.bio || null;
                avatarCacheTime[username] = now;
                return avatarCache[username];
            } catch(e) { return avatarCache[username] || null; }
        }

        async function refreshAvatar(username) {
            delete avatarCache[username];
            delete bioCache[username];
            delete avatarCacheTime[username];
            return fetchAvatar(username);
        }

        function avatarHtml(avatar, name, size) {
            const cls = size === 'large' ? 'avatar-large' : size === 'msg' ? 'msg-avatar' : 'avatar';
            const phCls = size === 'large' ? 'avatar-large-placeholder' : size === 'msg' ? 'msg-avatar-placeholder' : 'avatar-placeholder';
            const initial = (name || '?')[0].toUpperCase();
            if (avatar) return '<img class="' + cls + '" src="' + avatar + '" alt="avatar" loading="lazy">';
            return '<div class="' + phCls + '">' + initial + '</div>';
        }
        
        const savedUser = localStorage.getItem('currentUser');
        if (savedUser) {
            try {
                currentUser = JSON.parse(savedUser);
                updateLastSeen();
                if (lastSeenInterval) clearInterval(lastSeenInterval);
                lastSeenInterval = setInterval(updateLastSeen, 30000);
                if (localStorage.getItem('notificationEnabled') === 'true') notificationPermission = true;
                startOnlineStatusUpdates();
                showUsersList();
            } catch(e) { showMain(); }
        } else {
            showMain();
        }
    </script>
</body>
</html>`;
}
