<?php
define('API_KEY', '8238009802:AAECyO7TTAWn5X63OoKDHXfy4n8ITzbRsMs');

$admin_id = "7255472205"; // O'zingizning telegram ID raqamingizni kiritasiz
$apiKey = "0mcy8"; // @BuyAPIBot dan olingan API kalitni kiritasiz

function bot($method, $datas = []) {
    $url = "https://api.telegram.org/bot" . API_KEY . "/" . $method;
    $ch = curl_init();
    curl_setopt($ch, CURLOPT_URL, $url);
    curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
    curl_setopt($ch, CURLOPT_POSTFIELDS, $datas);
    $res = curl_exec($ch);
    if (curl_error($ch)) {
        var_dump(curl_error($ch));
    } else {
        return json_decode($res);
    }
}

$update = json_decode(file_get_contents('php://input'));
$message = $update->message ?? null;
$callback = $update->callback_query ?? null;
$bot = bot('getme')->result->username;
$storage_file = __DIR__ . "/last_results.json";
$users_file = __DIR__ . "/users.txt";

if ($message) {
    $cid = $message->chat->id;
    $text = trim($message->text);

    if (!file_exists($users_file)) {
        file_put_contents($users_file, "");
    }
    $users = file($users_file, FILE_IGNORE_NEW_LINES);
    if (!in_array($cid, $users)) {
        file_put_contents($users_file, $cid . "\n", FILE_APPEND);
    }
    
    if ($cid == $admin_id) {

        if ($text == "/panel") {
            bot('sendMessage', [
                'chat_id' => $cid,
                'text' => "📊 Admin panel:\n/stat — Statistika\n/xabar matn_yozasiz — Xabar yuborish"
            ]);
            exit;
        }

        if ($text == "/stat") {
            $user_count = count(file($users_file, FILE_IGNORE_NEW_LINES));
            bot('sendMessage', [
                'chat_id' => $cid,
                'text' => "📊 Foydalanuvchilar soni: $user_count ta"
            ]);
            exit;
        }

        if (strpos($text, "/xabar ") === 0) {
            $msg = substr($text, 7);
            $users = file($users_file, FILE_IGNORE_NEW_LINES);
            $ok = 0;
            foreach ($users as $uid) {
                bot('sendMessage', [
                    'chat_id' => $uid,
                    'text' => $msg
                ]);
                $ok++;
            }
            bot('sendMessage', [
                'chat_id' => $cid,
                'text' => "✅ $ok ta foydalanuvchiga yuborildi."
            ]);
            exit;
        }
    }
    
    if ($text == "/start") {
        bot('sendMessage', [
            'chat_id' => $cid,
            'text' => "🎵 <b>Assalomu alaykum!</b> @$bot ga xush kelibsiz!\nBu bot orqali <b>istalgan musiqani</b> tez va qulay yuklab olishingiz mumkin:\n\n• 🎧 <b>Qo‘shiqlar</b> – nomi yoki ijrochisini yozish kifoya\n• 📜 Qo‘shiq matnini ko‘rish (agar mavjud bo‘lsa)\n• 🎼 Sifatli <b>MP3</b> yuklab olish\n\n🚀 <b>Boshlash uchun:</b>\n1. Qo‘shiq nomi yoki havolasini yuboring\n2. Bot sizga yuklab olish uchun tayyor faylni beradi\n\n😎 Menga shunchaki qo‘shiq nomini yozib yuboring!",
            'parse_mode' => 'HTML'
        ]);
    } else {
        $api_url = "https://s1380.surkhandc.uz/API/Music/?key=$apiKey&name=" . urlencode($text);
        $json = file_get_contents($api_url);
        $data = json_decode($json, true);

        if ($data['status'] && !empty($data['natija'])) {
            $all_data = [];
            if (file_exists($storage_file)) {
                $all_data = json_decode(file_get_contents($storage_file), true) ?: [];
            }
            $all_data[$cid] = $data['natija'];
            file_put_contents($storage_file, json_encode($all_data));

            $buttons = [];
            foreach ($data['natija'] as $index => $mus) {
                $buttons[] = ['text' => (string)($index + 1), 'callback_data' => "play|" . $index];
            }
            $keyboard = array_chunk($buttons, 5);

            $matn = '';
            foreach ($data['natija'] as $index => $mus) {
                $title = ($mus['artist'] ? $mus['artist'] . " - " : "") . $mus['sarlavha'];
                $matn .= "<b>" . ($index + 1) . ".</b> " . $title . "\n";
            }

            bot('sendMessage', [
                'chat_id' => $cid,
                'text' => "$matn",
                'parse_mode' => 'HTML',
                'reply_markup' => json_encode(['inline_keyboard' => $keyboard])
            ]);
        } else {
            bot('sendMessage', [
                'chat_id' => $cid,
                'text' => "<b>❌ Musiqa topilmadi!</b>",
                'parse_mode' => 'HTML'
            ]);
        }
    }
}

if ($callback) {
    $cid = $callback->message->chat->id;
    $data = $callback->data;

    if (strpos($data, "play|") === 0) {
        $index = explode("|", $data)[1];
        if (file_exists($storage_file)) {
            $all_data = json_decode(file_get_contents($storage_file), true);
            if (!empty($all_data[$cid][$index])) {
                $mus = $all_data[$cid][$index];
                bot('sendAudio', [
                    'chat_id' => $cid,
                    'audio' => $mus['musiqa_url'],
                    'caption' => "<b>🎵 Musiqa yuklab olish uchun👇 @$bot</b>",
                    'parse_mode' => 'HTML'
                ]);
            }
        }
        bot('answerCallbackQuery', [
            'callback_query_id' => $callback->id
        ]);
    }
}
