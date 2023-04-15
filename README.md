# AIWordMaster
from flask import Flask, request, abort
from linebot import LineBotApi, WebhookHandler
from linebot.exceptions import InvalidSignatureError
from linebot.models import MessageEvent, TextMessage, TextSendMessage

import os
import random

CHANNEL_ACCESS_TOKEN = "B9YyJGwnffRchymjQeqm5h7sKAj9XbxUpWvBNZyvrgC3pvqoqwjiAIwWv18e7XeE7gpcmbHRqvQogZ9BOfMo6EaAM0sLqG2vEmk8bEWWy++qJ23NYVDJd1QyjDocq3K+I79LUjRNAuwEtbNQzYtC0wdB04t89/1O/w1cDnyilFU="
CHANNEL_SECRET = "87611ee13e3374c910952bf0b0fb304b"

line_bot_api = LineBotApi(CHANNEL_ACCESS_TOKEN)
handler = WebhookHandler(CHANNEL_SECRET)

app = Flask(__name__)

quiz_data = [
    {"ja": "犬", "en": "dog"},
    {"ja": "猫", "en": "cat"},
    {"ja": "鳥", "en": "bird"},
    {"ja": "魚", "en": "fish"},
    # ... 他のデータ
]

# ユーザーのクイズ状態を保持するための辞書
quiz_state = {}

# 問題と選択肢を生成する関数
def generate_question():
    question = random.choice(quiz_data)
    choices = [q['en'] for q in random.sample(quiz_data, 5) if q != question]
    if question['en'] not in choices:
        choices.pop()
        choices.append(question['en'])
    random.shuffle(choices)
    return question, choices


@app.route("/callback", methods=["POST"])
def callback():
    # LINEの署名を検証
    signature = request.headers["X-Line-Signature"]

    # リクエストのボディを取得
    body = request.get_data(as_text=True)

    try:
        handler.handle(body, signature)
    except InvalidSignatureError:
        abort(400)

    return "OK"

@handler.add(MessageEvent, message=TextMessage)
def handle_message(event):
    user_message = event.message.text
    user_id = event.source.user_id

    if user_message.lower() == "start":
        question, choices = generate_question()
        quiz_state[user_id] = question
        reply_text = "問題: {}\n選択肢: {}\n答えを入力してください。".format(question['ja'], ', '.join(choices))
    elif user_id in quiz_state:
        question = quiz_state[user_id]
        if user_message.lower() in [q['en'].lower() for q in quiz_data] or user_message.lower() in [q['ja'].lower() for q in quiz_data]:
            if user_message.lower() == question['en'].lower() or user_message.lower() == question['ja'].lower():
                reply_text = "正解です！おめでとうございます！\n次の問題に進むには 'start' と入力してください。"
            else:
                reply_text = "残念、不正解です。\n正解は '{}' でした。\n次の問題に進むには 'start' と入力してください。".format(question['en'] if user_message in [q['ja'] for q in quiz_data] else question['ja'])
            del quiz_state[user_id]
        else:
            reply_text = "クイズを開始するには 'start' と入力してください。"

    line_bot_api.reply_message(
        event.reply_token,
        TextSendMessage(text=reply_text)
    )


if __name__ == "__main__":
    app.run()
