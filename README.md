/***************************************************************
 * CONFIG
 * LINE_CHANNEL_ID / LINE_CHANNEL_SECRET เก็บใน Script Properties
 *  module เพิ่มสิทธิให้ผู้ใช้ Trial ใน Customer Sheet  == * ACTIVATE TRIAL==
 ***************************************************************/
const SPREADSHEET_ID = "1h04xJqqITAcRjRvx4YwwBD_qTysxHrqgB5vsXoDhCvU";
const CUSTOMER_SHEET = "Customer";

const SESSION_TTL = 21600;   // 6 ชั่วโมง
const STATE_TTL = 1800;      // 30 นาที
const USER_CACHE_TTL = 180;  // 3 นาที
const APP_VERSION = "2026-10-17-r4";   // แสดงท้ายหน้า Profile ไว้เช็กว่า deploy ถูกเวอร์ชัน
const DISABLE_USER_CACHE = true;       // ช่วงทดสอบ: อ่าน Sheet สดทุกครั้ง (ทดสอบเสร็จเปลี่ยนเป็น false)
const USER_CACHE_PREFIX = "user3_"; // เปลี่ยนเลขเมื่อโครงสร้าง userData เปลี่ยน เพื่อล้าง cache เก่า
const PKG_CACHE_TTL = 600;   // 10 นาที

// เปิด Spreadsheet ครั้งเดียวต่อหนึ่ง execution (openById ช้า)
var _customerSheet = null;
function getCustomerSheet_() {
  if (!_customerSheet) {
    _customerSheet = SpreadsheetApp
      .openById(SPREADSHEET_ID)
      .getSheetByName(CUSTOMER_SHEET);
  }
  return _customerSheet;
}

function setupLineConfig() {
  PropertiesService.getScriptProperties().setProperties({
    LINE_CHANNEL_ID: "2011970357",
    LINE_CHANNEL_SECRET: "3b1d7bff4b87657716c458fee8278700"
  });
}

/***************************************************************
 * ROUTING
 ***************************************************************/
function doGet(e) {
  const p = (e && e.parameter) || {};

  if (p.code || p.error) {
    return handleLineCallback_(p);
  }

  // ทดสอบว่า URL /exec เปิดได้และเป็นเวอร์ชันไหน: <URL>?page=ping
  if (p.page === "ping") {
    return ContentService.createTextOutput("ok " + APP_VERSION);
  }

  const page = ["profile", "trial", "subscription"].indexOf(p.page) >= 0
    ? p.page
    : "profile";

  const session = getSession_(p.sid);

  if (session) {
    try {
      return renderPage_(
        page,
        session.lineUserId,
        session.displayName,
        p.sid
      );
    } catch (err) {
      Logger.log("renderPage_ error: " + err);
      return errorPage_(
        "[" + APP_VERSION + "] page=" + page + " : " +
        (err.message || String(err))
      );
    }
  }

  // ไม่ใช้ lineUserId/displayName จาก URL เป็นหลักฐานยืนยันตัวตน
  return loginPage_(page);
}

/***************************************************************
 * RENDER PAGE
 ***************************************************************/
function renderPage_(page, lineUserId, displayName, sid) {
  const scriptUrl = ScriptApp.getService().getUrl();

  // หน้าเลือกแพ็กเกจ: ถ้ายังไม่มีแถวใน Customer ให้สร้างก่อน (กรณีสมัครสมาชิกใหม่โดยไม่เคย Trial)
  if (page === "subscription") {
    try {
      ensureCustomerRow_(lineUserId, displayName);
    } catch (err) {
      Logger.log("ensureCustomerRow_ error: " + err);
      return errorPage_(err.message || String(err));
    }
  }

  const fileMap = {
    profile: "Index",
    trial: "Trial",
    subscription: "Luxury_Subscription"
  };

  const titleMap = {
    profile: "Profile",
    trial: "Go Go T-View Trial",
    subscription: "Package Selector"
  };

  let userData;

  if (page === "trial") {
    // หน้า Trial เดิมไม่จำเป็นต้องอ่านข้อมูลสมาชิกเพื่อแสดงหน้า
    userData = {
      status: "unchecked",
      lineUserId: lineUserId,
      displayName: displayName
    };
  } else {
    userData = getUserDataFromSheet(lineUserId, displayName);
  }

  const t = HtmlService.createTemplateFromFile(fileMap[page]);

  t.userData = userData;
  t.scriptUrl = scriptUrl;
  t.lineUserId = lineUserId;
  t.displayName = displayName;
  t.sid = sid || "";

  // ใช้กับไฟล์หน้าอื่นที่อ้างอิงตัวแปรนี้อยู่
  t.displayNameJson = JSON.stringify(displayName || "-")
    .replace(/</g, "\\u003c");

  // ลิงก์ข้ามหน้าใช้ "LINE Login redirect" เหมือนเวอร์ชันแรกที่เปิด Luxury ได้
  // (ไม่ใช้ลิงก์ ?page=...&sid=... เพราะเปิดแล้วเจอหน้า "ไม่สามารถเปิดไฟล์ได้")
  // LINE จะ redirect กลับมาที่ doGet พร้อม code แล้ว callback จะสร้าง session ใหม่และแสดงหน้าปลายทาง
  t.trialUrl = "";
  t.subUrl = "";
  t.profileUrl = scriptUrl;

  if (page === "profile") {
    const urls = buildLineLoginUrls_(["trial", "subscription"]);
    t.trialUrl = urls.trial;
    t.subUrl = urls.subscription;
  } else if (page === "subscription") {
    t.profileUrl = buildLineLoginUrls_(["profile"]).profile;
  }

  return t.evaluate()
    .setTitle(titleMap[page])
    .setXFrameOptionsMode(HtmlService.XFrameOptionsMode.ALLOWALL);
}

function pageUrl_(page, sid) {
  return ScriptApp.getService().getUrl() +
    "?page=" + encodeURIComponent(page) +
    "&sid=" + encodeURIComponent(sid);
}

/***************************************************************
 * SESSION
 ***************************************************************/
function createSession_(lineUserId, displayName) {
  const sid = Utilities.getUuid();

  CacheService.getScriptCache().put(
    "sess_" + sid,
    JSON.stringify({
      lineUserId: lineUserId,
      displayName: displayName
    }),
    SESSION_TTL
  );

  return sid;
}

function getSession_(sid) {
  if (!sid) return null;

  const raw = CacheService.getScriptCache().get("sess_" + sid);
  return raw ? JSON.parse(raw) : null;
}

/***************************************************************
 * LINE LOGIN (OAuth2 / OpenID Connect)
 ***************************************************************/
function getLineConfig_() {
  const props = PropertiesService.getScriptProperties();
  const id = props.getProperty("LINE_CHANNEL_ID");
  const secret = props.getProperty("LINE_CHANNEL_SECRET");

  if (!id || !secret) {
    throw new Error(
      "ยังไม่ได้ตั้งค่า LINE_CHANNEL_ID / LINE_CHANNEL_SECRET"
    );
  }

  return {
    id: id,
    secret: secret
  };
}

function buildLineLoginUrls_(nexts) {
  const cfg = getLineConfig_();
  const redirectUri = ScriptApp.getService().getUrl();
  const cache = CacheService.getScriptCache();
  const urls = {};
  const states = {};

  nexts.forEach(function (next) {
    if (["profile", "trial", "subscription"].indexOf(next) < 0) {
      throw new Error("หน้าปลายทางไม่ถูกต้อง");
    }

    // แต่ละลิงก์ใช้ state/nonce ของตัวเอง และใช้ได้ครั้งเดียว
    const nonce = Utilities.getUuid();
    states["state_" + nonce] = next;

    urls[next] =
      "https://access.line.me/oauth2/v2.1/authorize?" +
      [
        "response_type=code",
        "client_id=" + encodeURIComponent(cfg.id),
        "redirect_uri=" + encodeURIComponent(redirectUri),
        "state=" + encodeURIComponent(next + "." + nonce),
        "nonce=" + encodeURIComponent(nonce),
        "scope=" + encodeURIComponent("profile openid")
      ].join("&");
  });

  cache.putAll(states, STATE_TTL);   // เรียก cache ครั้งเดียวแทนหลายครั้ง

  return urls;
}

function buildLineLoginUrl_(next) {
  return buildLineLoginUrls_([next])[next];
}

function handleLineCallback_(p) {
  const T0 = Date.now();
  let lastMark = T0;
  const times = [];
  const mark_ = function (label) {
    const now = Date.now();
    times.push(label + "=" + (now - lastMark) + "ms");
    lastMark = now;
  };

  try {
    if (p.error) {
      throw new Error(
        "LINE Login ไม่สำเร็จ: " +
        (p.error_description || p.error)
      );
    }

    if (!p.code) {
      throw new Error("ไม่ได้รับรหัสยืนยันจาก LINE");
    }

    const parts = String(p.state || "").split(".");
    const next = parts[0];
    const nonce = parts.slice(1).join(".");
    const cache = CacheService.getScriptCache();

    if (
      ["profile", "trial", "subscription"].indexOf(next) < 0 ||
      !nonce ||
      cache.get("state_" + nonce) !== next
    ) {
      throw new Error(
        "ลิงก์หมดอายุหรือถูกใช้ไปแล้ว กรุณาเข้าสู่ระบบใหม่"
      );
    }

    cache.remove("state_" + nonce);

    const cfg = getLineConfig_();
    const redirectUri = ScriptApp.getService().getUrl();

    const tokenResponse = UrlFetchApp.fetch(
      "https://api.line.me/oauth2/v2.1/token",
      {
        method: "post",
        muteHttpExceptions: true,
        contentType: "application/x-www-form-urlencoded",
        payload: {
          grant_type: "authorization_code",
          code: p.code,
          redirect_uri: redirectUri,
          client_id: cfg.id,
          client_secret: cfg.secret
        }
      }
    );

    const tokenData = JSON.parse(
      tokenResponse.getContentText()
    );

    mark_("token");

    if (
      tokenResponse.getResponseCode() !== 200 ||
      !tokenData.id_token
    ) {
      throw new Error(
        "ขอ token จาก LINE ไม่สำเร็จ: " +
        (tokenData.error_description ||
          tokenData.error ||
          tokenResponse.getResponseCode())
      );
    }

    // id_token ได้มาจาก token endpoint ของ LINE โดยตรง (server-to-server, HTTPS, ใช้ client_secret)
    // จึงถอดรหัส payload ในเครื่องแล้วตรวจ iss/aud/exp/nonce เอง ไม่ต้องยิง /verify อีกรอบ (ประหยัด 1 network call)
    let identity;
    try {
      identity = JSON.parse(
        Utilities.newBlob(
          Utilities.base64DecodeWebSafe(
            String(tokenData.id_token).split(".")[1]
          )
        ).getDataAsString("UTF-8")
      );
    } catch (e) {
      throw new Error("อ่าน id_token จาก LINE ไม่ได้");
    }

    if (
      identity.iss !== "https://access.line.me" ||
      String(identity.aud) !== String(cfg.id) ||
      Number(identity.exp) * 1000 < Date.now()
    ) {
      throw new Error("ตรวจสอบ id_token ของ LINE ไม่ผ่าน");
    }

    if (String(identity.nonce || "") !== nonce) {
      throw new Error("ตรวจสอบ nonce ของ LINE Login ไม่ผ่าน");
    }

    mark_("decode");

    const lineUserId = String(identity.sub || "").trim();
    let displayName = String(identity.name || "").trim();

    if (!lineUserId) {
      throw new Error("LINE ไม่ส่ง User ID กลับมา");
    }

    // บางกรณี name ไม่มีใน id_token: อ่าน profile ด้วย access token
    if (!displayName && tokenData.access_token) {
      const profileResponse = UrlFetchApp.fetch(
        "https://api.line.me/v2/profile",
        {
          method: "get",
          muteHttpExceptions: true,
          headers: {
            Authorization: "Bearer " + tokenData.access_token
          }
        }
      );

      if (profileResponse.getResponseCode() === 200) {
        const profile = JSON.parse(
          profileResponse.getContentText()
        );

        if (String(profile.userId || "") === lineUserId) {
          displayName = String(profile.displayName || "").trim();
        }
      }
    }

    if (!displayName) {
      throw new Error("LINE ไม่ส่ง DisplayName กลับมา");
    }

    // Login ไม่สร้างแถวใหม่ใน Customer
    // ผู้ใช้เดิม: readUserFromSheet_ จะอัปเดต DisplayName ล่าสุดให้ในรอบอ่านเดียวกัน
    clearUserCache_(lineUserId);

    const sid = createSession_(lineUserId, displayName);
    mark_("session");

    const page = renderPage_(
      next,
      lineUserId,
      displayName,
      sid
    );

    mark_("render");
    Logger.log(
      "[timing] total=" + (Date.now() - T0) + "ms | " + times.join(" ")
    );

    return page;

  } catch (err) {
    Logger.log("handleLineCallback_ error: " + err);
    return errorPage_(err.message || String(err));
  }
}

/***************************************************************
 * CUSTOMER ROW HELPERS
 * Customer: A = Customer_ID, B = Name, C = LineUserID,
 *           D = DisplayName, M = FirstSeen
 ***************************************************************/

// คำนวณ Customer_ID ถัดไปจากแถวล่าสุดที่เป็นรูปแบบ C + ตัวเลข (เช่น C005 -> C006)
function nextCustomerId_(sheet) {
  const lastRow = sheet.getLastRow();
  let nextIdNum = 1;
  let idPadding = 3;

  if (lastRow > 1) {
    const colA = sheet.getRange(2, 1, lastRow - 1, 1).getValues();

    for (let i = colA.length - 1; i >= 0; i--) {
      const m = String(colA[i][0]).trim().match(/^C(\d+)$/i);

      if (m) {
        nextIdNum = parseInt(m[1], 10) + 1;
        idPadding = m[1].length;
        break;
      }
    }
  }

  return "C" + String(nextIdNum).padStart(idPadding, "0");
}

// ตรวจว่ามี LineUserID/DisplayName ใน Customer หรือยัง
// - มีแล้ว: ไม่ทำอะไร (return created=false)
// - ยังไม่มี: เพิ่มแถวใหม่ A,B,C,D,M แล้ว return created=true
function ensureCustomerRow_(lineUserId, displayName) {
  const lock = LockService.getScriptLock();
  lock.waitLock(15000);

  try {
    const sheet = getCustomerSheet_();

    if (!sheet) {
      throw new Error("ไม่พบ Sheet ชื่อ 'Customer'");
    }

    const row = findRowByLineUserId_(sheet, lineUserId);

    if (row) {
      // มีแล้ว: เติม DisplayName ถ้าว่างอยู่
      if (!String(sheet.getRange(row, 4).getValue()).trim()) {
        sheet.getRange(row, 4).setValue(displayName);
        clearUserCache_(lineUserId);
      }
      return { created: false, row: row };
    }

    const newRow = sheet.getLastRow() + 1;
    const tz = Session.getScriptTimeZone();
    const firstSeen = Utilities.formatDate(new Date(), tz, "dd/MM/yyyy");

    const rowData = new Array(28).fill("");
    rowData[0] = nextCustomerId_(sheet);   // A Customer_ID
    rowData[1] = "User";                   // B Name
    rowData[2] = lineUserId;               // C LineUserID
    rowData[3] = displayName;              // D DisplayName
    rowData[12] = firstSeen;               // M FirstSeen (dd/mm/yyyy)

    // บังคับคอลัมน์ M เป็นข้อความ เพื่อไม่ให้ Sheet แปลงเป็นวันที่
    sheet.getRange(newRow, 13).setNumberFormat("@");
    sheet.getRange(newRow, 1, 1, rowData.length).setValues([rowData]);

    clearUserCache_(lineUserId);
    return { created: true, row: newRow };

  } finally {
    lock.releaseLock();
  }
}

/***************************************************************
 * LOGIN / ERROR PAGE
 ***************************************************************/
function loginPage_(next) {
  const url = buildLineLoginUrl_(next);

  // สำหรับหน้า profile พยายามพาไป LINE Login อัตโนมัติ
  // คงปุ่มไว้ด้วย เพราะบาง WebView อาจบล็อกการนำทางอัตโนมัติ
  const autoRedirect = next === "profile"
    ? "<script>window.top.location.replace(" +
      JSON.stringify(url).replace(/</g, "\\u003c") +
      ");</script>"
    : "";

  return HtmlService.createHtmlOutput(`<!DOCTYPE html>
<html lang="th">
<head>
  <meta charset="UTF-8">
  <meta name="viewport"
        content="width=device-width, initial-scale=1.0">
  <title>เข้าสู่ระบบ</title>
  <style>
    body {
      font-family: 'Kanit', sans-serif;
      background: #1e1310;
      color: #fde68a;
      min-height: 100vh;
      margin: 0;
      display: flex;
      align-items: center;
      justify-content: center;
      padding: 24px;
    }
    .box {
      background: #2b1b17;
      border: 1px solid #5c3a31;
      border-radius: 32px;
      padding: 48px 36px;
      max-width: 640px;
      width: 100%;
      text-align: center;
    }
    h1 {
      font-size: 34px;
      margin: 0 0 12px;
    }
    p {
      color: #d6b98c;
      font-size: 20px;
      margin: 0 0 32px;
    }
    a {
      display: block;
      background: #06C755;
      color: #fff;
      text-decoration: none;
      font-size: 26px;
      font-weight: 600;
      padding: 22px;
      border-radius: 20px;
    }
  </style>
</head>
<body>
  <div class="box">
    <h1>เข้าสู่ระบบ</h1>
    <p>กรุณาเข้าสู่ระบบด้วย LINE เพื่อดำเนินการต่อ</p>
    <a href="${escapeHtml_(url)}"
       target="_top">เข้าสู่ระบบด้วย LINE</a>
  </div>
  ${autoRedirect}
</body>
</html>`)
    .setTitle("เข้าสู่ระบบ")
    .setXFrameOptionsMode(
      HtmlService.XFrameOptionsMode.ALLOWALL
    );
}

function errorPage_(message) {
  const url = ScriptApp.getService().getUrl();

  return HtmlService.createHtmlOutput(`<!DOCTYPE html>
<html lang="th">
<head>
  <meta charset="UTF-8">
  <meta name="viewport"
        content="width=device-width, initial-scale=1.0">
  <title>ข้อผิดพลาด</title>
  <style>
    body {
      font-family: 'Kanit', sans-serif;
      background: #1e1310;
      color: #fde68a;
      min-height: 100vh;
      margin: 0;
      display: flex;
      align-items: center;
      justify-content: center;
      padding: 24px;
    }
    .box {
      background: #2b1b17;
      border: 1px solid #7f1d1d;
      border-radius: 32px;
      padding: 48px 36px;
      max-width: 640px;
      width: 100%;
      text-align: center;
    }
    h1 {
      color: #f87171;
      font-size: 32px;
      margin: 0 0 16px;
    }
    p {
      font-size: 20px;
      color: #d6b98c;
      margin: 0 0 32px;
      word-break: break-word;
    }
    a {
      display: inline-block;
      background: #2563eb;
      color: #fff;
      text-decoration: none;
      font-size: 22px;
      padding: 16px 36px;
      border-radius: 16px;
    }
  </style>
</head>
<body>
  <div class="box">
    <h1>ข้อผิดพลาด</h1>
    <p>${escapeHtml_(message)}</p>
    <a href="${escapeHtml_(url)}"
       target="_top">กลับหน้าแรก</a>
  </div>
</body>
</html>`)
    .setTitle("ข้อผิดพลาด")
    .setXFrameOptionsMode(
      HtmlService.XFrameOptionsMode.ALLOWALL
    );
}

function escapeHtml_(s) {
  return String(s)
    .replace(/&/g, "&amp;")
    .replace(/</g, "&lt;")
    .replace(/>/g, "&gt;")
    .replace(/"/g, "&quot;")
    .replace(/'/g, "&#39;");
}

/***************************************************************
 * CUSTOMER DATA
 ***************************************************************/
function clearUserCache_(lineUserId) {
  CacheService.getScriptCache()
    .remove(USER_CACHE_PREFIX + lineUserId);
}

function getUserDataFromSheet(
  lineUserId,
  displayNameFromLogin
) {
  const cache = CacheService.getScriptCache();
  const key = USER_CACHE_PREFIX + lineUserId;
  const hit = DISABLE_USER_CACHE ? null : cache.get(key);

  if (hit) {
    return JSON.parse(hit);
  }

  const data = readUserFromSheet_(
    lineUserId,
    displayNameFromLogin
  );

  if (!data.error && !DISABLE_USER_CACHE) {
    cache.put(
      key,
      JSON.stringify(data),
      USER_CACHE_TTL
    );
  }

  return data;
}

function findRowByLineUserId_(sheet, lineUserId) {
  const lastRow = sheet.getLastRow();

  if (lastRow < 2) {
    return 0;
  }

  const found = sheet
    .getRange(2, 3, lastRow - 1, 1)
    .createTextFinder(String(lineUserId).trim())
    .matchEntireCell(true)
    .matchCase(true)
    .findNext();

  return found ? found.getRow() : 0;
}

function readUserFromSheet_(
  lineUserId,
  displayNameFromLogin
) {
  // สิทธิปุ่ม (ตามเงื่อนไข):
  //  - ไม่มีแถวใน Customer      -> สมัครใหม่ + Trial เปิด (ต่ออายุ ปิด)
  //  - มีแถว, PlanSubscript ว่าง -> สมัครใหม่ + Trial เปิด (ต่ออายุ ปิด)
  //  - มีแถว, PlanSubscript มีค่า -> ต่ออายุ เปิด เท่านั้น
  const dbg = { ver: APP_VERSION, ss: "-", row: 0, rawF: "-" };

  const noMember = function (name, err, hasRow) {
    return {
      status: "no_member",
      lineUserId: lineUserId,
      displayName:
        name || displayNameFromLogin || "-",
      error: !!err,
      canRenew: false,
      canRegister: !err,
      canTrial: !err,
      debug: dbg
    };
  };

  try {
    const sheet = getCustomerSheet_();

    if (!sheet) {
      return noMember(null, true);
    }

    dbg.ss = "…" + SPREADSHEET_ID.slice(-6);

    const row = findRowByLineUserId_(
      sheet,
      lineUserId
    );

    dbg.row = row;

    if (!row) {
      return noMember();
    }

    const cols = Math.min(
      28,
      sheet.getMaxColumns()
    );

    const r = sheet
      .getRange(row, 1, 1, cols)
      .getValues()[0];

    // อัปเดต DisplayName ล่าสุดจาก LINE (เขียนเฉพาะเมื่อเปลี่ยน)
    if (
      displayNameFromLogin &&
      displayNameFromLogin !== "-" &&
      String(r[3]) !== displayNameFromLogin
    ) {
      sheet.getRange(row, 4).setValue(displayNameFromLogin);
      r[3] = displayNameFromLogin;
    }

    dbg.rawF = JSON.stringify(r[5]);

    const usedTrial = String(
      r[4] || ""
    ).trim();

    const planSubscript = String(
      r[5] || ""
    ).trim();

    // แถว C/D ที่สร้างตอน Login ยังไม่ถือเป็นสมาชิก
    // แต่ Trial ที่ UsedTrial = Y ถือว่าเป็นสมาชิก
    if (!planSubscript && !usedTrial) {
      return noMember(r[3], false, true);
    }

    const formatDate = function (v) {
      if (!v) return "-";

      const d = new Date(v);

      if (isNaN(d.getTime())) {
        return String(v);
      }

      return String(d.getDate()).padStart(2, "0") +
        "/" +
        String(d.getMonth() + 1).padStart(2, "0") +
        "/" +
        (d.getFullYear() + 543);
    };

    return {
      status: "active",
      lineUserId: lineUserId,
      displayName:
        r[3] || displayNameFromLogin || "-",
      customerId: r[0] || "-",
      planSubscript:
        planSubscript || "Trial",
      memberStatus:
        r[10] || "active",
      maxSymbol:
        r[6] || "-",
      indyMaxFollow:
        r[21] || "-",
      startDate:
        formatDate(r[8]),
      expireDate:
        formatDate(r[9]),
      usedTrial: usedTrial,
      hasPlan: !!planSubscript,
      canRenew: !!planSubscript,
      canRegister: !planSubscript,
      canTrial: !planSubscript,
      debug: dbg
    };

  } catch (error) {
    Logger.log(
      "readUserFromSheet_ error: " + error
    );

    return noMember(null, true);
  }
}

/***************************************************************
 * PACKAGE LIST
 * อ่าน packageName จาก PricePackage
 * เรียงตาม sortOrder และแสดงเฉพาะ active = Y
 ***************************************************************/
function getPackageList() {
  // ใช้ openById เหมือนฟังก์ชันอื่นในระบบ (อย่าใช้ getActiveSpreadsheet)
  const sheet = SpreadsheetApp
    .openById(SPREADSHEET_ID)
    .getSheetByName(PAYMENT_CONFIG.PACKAGE_SHEET);   // 'PricePackage'
  if (!sheet) throw new Error('ไม่พบ Sheet ' + PAYMENT_CONFIG.PACKAGE_SHEET);

  const values = sheet.getDataRange().getValues();
  if (values.length < 2) return [];

  // หาตำแหน่งคอลัมน์จาก header
  const headers = values[0].map(h => String(h).trim());
  const idx = {};
  headers.forEach((h, i) => { idx[h] = i; });

  if (idx.planCode === undefined) {
    throw new Error('PricePackage ไม่มี Header: planCode');
  }

  // **** จุดสำคัญ: คืนค่า planCode ไม่ใช่ packageName ****
  return values.slice(1)
    .map(row => ({
      planCode:  String(row[idx.planCode] || '').trim(),
      active:    idx.active !== undefined
                   ? String(row[idx.active] || '').trim().toUpperCase()
                   : 'Y',
      sortOrder: idx.sortOrder !== undefined
                   ? (Number(row[idx.sortOrder]) || 0)
                   : 0
    }))
    .filter(p => p.planCode !== '' && p.active === 'Y')   // เฉพาะแพ็กเกจที่เปิดใช้
    .sort((a, b) => a.sortOrder - b.sortOrder)            // เรียงตาม sortOrder
    .map(p => p.planCode);
}
/*************************END GetpackageList******************************** */

/***************************************************************
 * ACTIVATE TRIAL
 ***************************************************************/
function activateTrialBackend(sid) {
  const lock = LockService.getScriptLock();
  let lockHeld = false;

  try {
    const session = getSession_(sid);

    if (!session) {
      throw new Error(
        "Session หมดอายุ กรุณาเข้าสู่ระบบใหม่"
      );
    }

    const lineUserId = session.lineUserId;
    const displayName = session.displayName;

    lock.waitLock(15000);
    lockHeld = true;

    const sheet = getCustomerSheet_();

    if (!sheet) {
      throw new Error(
        "ไม่พบ Sheet ชื่อ 'Customer'"
      );
    }

    const existingRow = findRowByLineUserId_(
      sheet,
      lineUserId
    );

    // มีเพียงแถวข้อมูล C/D ยังสมัคร Trial ได้
    if (existingRow) {
      const existing = sheet
        .getRange(existingRow, 1, 1, 11)
        .getValues()[0];

      const usedTrial = String(
        existing[4] || ""
      ).trim();

      const plan = String(
        existing[5] || ""
      ).trim();

      if (usedTrial || plan) {
        throw new Error(
          "บัญชีนี้มีข้อมูลสมาชิกหรือเคยทดลองใช้แล้ว"
        );
      }
    }

    // หา Customer ID ถัดไปจากคอลัมน์ A
    const lastRow = sheet.getLastRow();
    let nextIdNum = 1;
    let idPadding = 3;

    if (lastRow > 1) {
      const colA = sheet
        .getRange(2, 1, lastRow - 1, 1)
        .getValues();

      for (
        let i = colA.length - 1;
        i >= 0;
        i--
      ) {
        const m = String(
          colA[i][0]
        ).trim().match(/^C(\d+)$/);

        if (m) {
          nextIdNum =
            parseInt(m[1], 10) + 1;
          idPadding = m[1].length;
          break;
        }
      }
    }

    const newCustomerId =
      "C" +
      String(nextIdNum).padStart(
        idPadding,
        "0"
      );

    const today = new Date();
    const expire = new Date();
    expire.setMonth(
      expire.getMonth() + 1
    );

    const tz = Session.getScriptTimeZone();

    const startDateStr =
      Utilities.formatDate(
        today,
        tz,
        "dd-MM-yyyy"
      );

    const expireDateStr =
      Utilities.formatDate(
        expire,
        tz,
        "dd-MM-yyyy"
      );

    const rowData =
      new Array(28).fill("");

    rowData[0] = newCustomerId;
    rowData[1] = "User";
    rowData[2] = lineUserId;
    rowData[3] = displayName;
    rowData[4] = "Y";  // Trial
    rowData[6] = 30;  // MaxSymbol
    rowData[8] = startDateStr;
    rowData[9] = expireDateStr;
    rowData[10] = "active";
    rowData[12] = startDateStr;
    rowData[17] = 20;  //คือ field ที่ให้โควต้าเตือนกี่ตัว (PriceAlertLimit) จากหุ้นใน watchlist
    rowData[21] = 4; // IndyMaxFollow ที่ยอมให้ติดตาม
    rowData[22] = "xxx@gmail.com";
    rowData[27] = "FALSE";

    if (existingRow) {
      // ใช้แถวที่ LINE Login สร้างไว้
      sheet
        .getRange(
          existingRow,
          1,
          1,
          rowData.length
        )
        .setValues([rowData]);
    } else {
      // รองรับกรณีข้อมูลเก่าที่ยังไม่มีแถว
      sheet.appendRow(rowData);
    }

    clearUserCache_(lineUserId);

    return {
      success: true,
      customerId: newCustomerId,
      startDate: startDateStr,
      expireDate: expireDateStr
    };

  } catch (error) {
    Logger.log(
      "activateTrialBackend error: " +
      error.toString()
    );

    return {
      success: false,
      error:
        error.message ||
        error.toString()
    };

  } finally {
    if (lockHeld) {
      try {
        lock.releaseLock();
      } catch (e) {
        Logger.log(
          "releaseLock error: " + e
        );
      }
    }
  }
}
