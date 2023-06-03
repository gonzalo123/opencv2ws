## Live stream over WebSocket with Python and OpenCV

In this article, we will conduct a small experiment. We will stream the live feed from a camera (for example, my laptop's camera) and transmit it using WebSockets to a web browser. In a real-life application, I would use a Django application, an MQTT broker as an event bus, and pure WebSockets with Django Channels. However, here we will use a minimal Flask application and WebSockets with the Socket.io library. Here is the application:



```python
import cv2
from flask import Flask, render_template
from flask_socketio import SocketIO
from datetime import datetime

app = Flask(__name__)
socketio = SocketIO(app)


@app.route('/')
def index():
    return render_template('index.html')


def video_stream():
    face_cascade = cv2.CascadeClassifier(cv2.data.haarcascades + 'haarcascade_frontalface_default.xml')
    video_capture = cv2.VideoCapture(0)

    while True:
        success, frame = video_capture.read()

        if success:
            gray = cv2.cvtColor(frame, cv2.COLOR_BGR2GRAY)
            faces = face_cascade.detectMultiScale(gray, scaleFactor=1.1, minNeighbors=5, minSize=(30, 30))
            current_time = datetime.now().strftime("%H:%M:%S")
            cv2.putText(frame, current_time, (10, 30), cv2.FONT_HERSHEY_SIMPLEX, 1, (0, 255, 0), 2)

            for (x, y, w, h) in faces:
                cv2.rectangle(frame, (x, y), (x + w, y + h), (0, 255, 0), 2)
            _, encoded_frame = cv2.imencode('.jpg', frame)
            frame_data = encoded_frame.tobytes()
            socketio.emit('frame', frame_data)

    video_capture.release()


socketio.start_background_task(video_stream)
socketio.run(app, allow_unsafe_werkzeug=True)
```

The video_stream() function captures video frames (in a background task) from the default camera and performs face detection using the Haar cascade classifier from OpenCV. It continuously reads frames, converts them to grayscale, detects faces, and draws rectangles around the detected faces. The frames are encoded as JPEG images and sent to the clients connected via SocketIO.

And finally the cliente part:

```html
<html>
<head>
    <title>Video Streaming Example</title>
    <script src="//cdnjs.cloudflare.com/ajax/libs/socket.io/4.4.0/socket.io.js"></script>
</head>
<body>
<img id="video-frame" src=""/>
<script type="text/javascript">
    (function () {
        let socket = io();
        let img = document.getElementById('video-frame');

        socket.on('frame', function (frameData) {
            let byteArray = new Uint8Array(frameData);
            let binaryData = '';
            for (var i = 0; i < byteArray.length; i++) {
                binaryData += String.fromCharCode(byteArray[i]);
            }
            img.src = 'data:image/jpeg;base64,' + btoa(binaryData);
        });
    })();
</script>
</body>
</html>
```

Here, we obtain the binary data of the image from the WebSocket, convert it to base64, and place it in the src attribute of an image.

And that's all. Our live stream from a camera in our browser.