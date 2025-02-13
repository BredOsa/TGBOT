#include <tgbot/tgbot.h>
#include <cpr/cpr.h>
#include <mysqlx/xdevapi.h>
#include <iostream>
#include <windows.h>
#include <ctime>
#include <openssl/hmac.h>

using namespace TgBot;
using namespace mysqlx;

const std::string API_KEY = "API key";
const std::string API_SECRET = "API sec";
const std::string DB_HOST = "localhost";
const int DB_PORT = 33060;
const std::string DB_USER = "root";
const std::string DB_PASS = "1228";
const std::string DB_NAME = "bybit_bot";
const std::string REFERRAL_LINK = "https://www.bybit.com/invite?ref=39YDMN";

std::string to_utf8(const std::string& str) {
    int size_needed = MultiByteToWideChar(CP_ACP, 0, str.c_str(), -1, NULL, 0);
    if (size_needed <= 0) return "";
    std::wstring wstr(size_needed, 0);
    MultiByteToWideChar(CP_ACP, 0, str.c_str(), -1, &wstr[0], size_needed);

    size_needed = WideCharToMultiByte(CP_UTF8, 0, wstr.c_str(), -1, NULL, 0, NULL, NULL);
    if (size_needed <= 0) return "";
    std::string utf8str(size_needed, 0);
    WideCharToMultiByte(CP_UTF8, 0, wstr.c_str(), -1, &utf8str[0], size_needed, NULL, NULL);

    return utf8str;
}

std::string generateSignature(const std::string& queryString) {
    unsigned char* digest;
    digest = HMAC(EVP_sha256(), API_SECRET.c_str(), API_SECRET.length(),
        (unsigned char*)queryString.c_str(), queryString.length(), NULL, NULL);
    char mdString[65];
    for (int i = 0; i < 32; i++)
        sprintf_s(&mdString[i * 2], sizeof(mdString) - i * 2, "%02x", (unsigned int)digest[i]);
    return std::string(mdString);
}

bool isRegisteredInBybit(const std::string& uid) {
    std::time_t now = std::time(0);
    std::string timestamp = std::to_string(now * 1000);
    std::string queryString = "api_key=" + API_KEY + "&timestamp=" + timestamp + "&uid=" + uid;
    std::string signature = generateSignature(queryString);

    auto response = cpr::Get(cpr::Url{ "https://api.bybit.com/v5/user/referred-list" },
        cpr::Header{
            {"X-BYBIT-API-KEY", API_KEY},
            {"X-BYBIT-SIGN", signature},
            {"X-BYBIT-TIMESTAMP", timestamp}
        },
        cpr::Parameters{
            {"api_key", API_KEY},
            {"timestamp", timestamp},
            {"sign", signature},
            {"uid", uid}
        });

    std::cout << "HTTP Status Code: " << response.status_code << std::endl;
    std::cout << "Bybit API Response: " << response.text << std::endl;

    if (response.status_code == 200 && response.text.find(uid) != std::string::npos) {
        return true;
    }
    return false;
}

bool isRegistered(const std::string& uid) {
    try {
        Session session(DB_HOST, DB_PORT, DB_USER, DB_PASS);
        Schema db = session.getSchema(DB_NAME);
        Table referrals = db.getTable("referrals");
        RowResult res = referrals.select("uid").where("uid = :uid").bind("uid", uid).execute();
        return res.count() > 0;
    }
    catch (const mysqlx::Error& err) {
        std::cerr << "MySQL error: " << err.what() << std::endl;
        return false;
    }
}

void saveToDatabase(const std::string& uid) {
    try {
        Session session(DB_HOST, DB_PORT, DB_USER, DB_PASS);
        Schema db = session.getSchema(DB_NAME);
        Table referrals = db.getTable("referrals");
        referrals.insert("uid", "referred").values(uid, true).execute();
    }
    catch (const mysqlx::Error& err) {
        std::cerr << "MySQL error: " << err.what() << std::endl;
    }
}

int main() {
    Bot bot("TGBOT TOKEN");
    bot.getEvents().onCommand("start", [&bot](Message::Ptr message) {
        bot.getApi().sendMessage(message->chat->id, to_utf8("Привет! Отправь мне свой UID для проверки регистрации."));
        });

    bot.getEvents().onAnyMessage([&bot](Message::Ptr message) {
        if (message->text.empty() || message->text[0] == '/') return;
        std::string uid = message->text;

        if (isRegistered(uid)) {
            bot.getApi().sendMessage(message->chat->id, to_utf8(" Вы уже зарегистрированы!"));
        }
        else if (isRegisteredInBybit(uid)) {
            saveToDatabase(uid);
            bot.getApi().sendMessage(message->chat->id, to_utf8(" Вы зарегистрированы по нашей реферальной ссылке!"));
        }
        else {
            bot.getApi().sendMessage(message->chat->id, to_utf8(" Вы не зарегистрированы! Пожалуйста, зарегистрируйтесь по ссылке: " + REFERRAL_LINK));
        }
        });

    try {
        TgBot::TgLongPoll longPoll(bot);
        while (true) {
            longPoll.start();
        }
    }
    catch (std::exception& e) {
        std::cerr << "Error: " << e.what() << std::endl;
    }
    return 0;
}
