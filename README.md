Article Outline
1. Project Initialization

Initializing Next.js
Installing necessary dependencies such as openai
2. Creating the `useRecordVoice` Hook

Developing a hook for voice recording
Utilizing the MediaRecorder API
3. Creating the Button Component

Building a button component for controlling voice recording
4. Checking Functionality

Running the project and verifying essential functions
5. Setting up the Speech Recognition API

Creating an API endpoint for processing audio data and interacting with the OpenAI API
6. Additional Features and Enhancements

Developing helper functions for audio and base64 conversion
Implementing additional features to obtain textual output
1. Project Initialization
To start, initialize a Next.js project and install necessary dependencies, including openai.

Initializing a Next.js project

npx create-next-app my-voice-app
Installing necessary dependencies ( openai v4, dotenv )

npm install openai dotenv
2. Creating the useRecordVoice Hook
Create a hook for convenient voice recording and use the MediaRecorder API.

import { useEffect, useState, useRef } from "react";

export const useRecordVoice = () => {
  // State to hold the media recorder instance
  const [mediaRecorder, setMediaRecorder] = useState(null);

  // State to track whether recording is currently in progress
  const [recording, setRecording] = useState(false);

  // Ref to store audio chunks during recording
  const chunks = useRef([]);

  // Function to start the recording
  const startRecording = () => {
    if (mediaRecorder) {
      mediaRecorder.start();
      setRecording(true);
    }
  };

  // Function to stop the recording
  const stopRecording = () => {
    if (mediaRecorder) {
      mediaRecorder.stop(); 
      setRecording(false);
    }
  };

  // Function to initialize the media recorder with the provided stream
  const initialMediaRecorder = (stream) => {
    const mediaRecorder = new MediaRecorder(stream);

    // Event handler when recording starts
    mediaRecorder.onstart = () => {
      chunks.current = []; // Resetting chunks array
    };

    // Event handler when data becomes available during recording
    mediaRecorder.ondataavailable = (ev) => {
      chunks.current.push(ev.data); // Storing data chunks
    };

    // Event handler when recording stops
    mediaRecorder.onstop = () => {
      // Creating a blob from accumulated audio chunks with WAV format
      const audioBlob = new Blob(chunks.current, { type: "audio/wav" });
      console.log(audioBlob, 'audioBlob')

      // You can do something with the audioBlob, like sending it to a server or processing it further
    };

    setMediaRecorder(mediaRecorder);
  };

  useEffect(() => {
    if (typeof window !== "undefined") {
      navigator.mediaDevices
        .getUserMedia({ audio: true })
        .then(initialMediaRecorder);
    }
  }, []); 

  return { recording, startRecording, stopRecording
 };
};
3. Creating the Button Component
Develop a button component for starting and stopping voice recording.

"use client";

import { useRecordVoice } from "@/hooks/useRecordVoice";
import { IconMicrophone } from "@/app/components/IconMicrophone";

const Microphone = () => {
  const { startRecording, stopRecording } = useRecordVoice();

  return (
    // Button for starting and stopping voice recording
    <button
      onMouseDown={startRecording}    // Start recording when mouse is pressed
      onMouseUp={stopRecording}        // Stop recording when mouse is released
      onTouchStart={startRecording}    // Start recording when touch begins on a touch device
      onTouchEnd={stopRecording}        // Stop recording when touch ends on a touch device
      className="border-none bg-transparent w-10"
    >
      {/* Microphone icon component */}
      <IconMicrophone />
    </button>
  );
};

export { Microphone };
4. Checking Functionality
Run the project and check basic functionality.

npm run dev
Open http://localhost:3000/ in your browser and ensure that the microphone permission prompt appears.


Next, the microphone indicator should light up in your browser tab, indicating that you did everything right.


Next, hold down the microphone and say anything, after which you should see something similar to this in the console, only yours may be different in size.


5. Setting up the Speech Recognition API
Create an API endpoint for processing audio data and interacting with the OpenAI API.

In the src/app/api/speechToText/route.js folder, I will create the speechToText api.

import { NextResponse } from "next/server";
import fs from "fs";
import * as dotenv from "dotenv"; 
import OpenAI from "openai";
import { env } from "../../config/env";

dotenv.config();

const openai = new OpenAI({
  apiKey: env.OPENAI_API_KEY,
});

export async function POST(req) {
  const body = await req.json();

  const base64Audio = body.audio;

  // Convert the base64 audio data to a Buffer
  const audio = Buffer.from(base64Audio, "base64");

  // Define the file path for storing the temporary WAV file
  const filePath = "tmp/input.wav";

  try {
    // Write the audio data to a temporary WAV file synchronously
    fs.writeFileSync(filePath, audio);

    // Create a readable stream from the temporary WAV file
    const readStream = fs.createReadStream(filePath);

    const data = await openai.audio.transcriptions.create({
      file: readStream,
      model: "whisper-1",
    });

    // Remove the temporary file after successful processing
    fs.unlinkSync(filePath);

    return NextResponse.json(data);
  } catch (error) {
    console.error("Error processing audio:", error);
    return NextResponse.error();
  }
}
In short, what happens here is that we create a file into which we write base64Audio and then read it and, after a successful request, delete the file.

Also, be sure to create a tmp folder on the same level as src and add .gitignore so that you can commit this folder and push.

6. Additional Features and Enhancements
Add helper functions for audio and base64 conversion and improve functionality.

blobToBase64

//callback - where we want to get result
const blobToBase64 = (blob, callback) => {
  const reader = new FileReader();
  reader.onload = function () {
    const base64data = reader?.result?.split(",")[1];
    callback(base64data);
  };
  reader.readAsDataURL(blob);
};

export { blobToBase64 };
createMediaStream

// Function to calculate the peak level from the analyzer data
const getPeakLevel = (analyzer) => {
  // Create a Uint8Array to store the audio data
  const array = new Uint8Array(analyzer.fftSize);

  // Get the time domain data from the analyzer and store it in the array
  analyzer.getByteTimeDomainData(array);

  // Calculate the peak level by finding the maximum absolute deviation from 127
  return (
    array.reduce((max, current) => Math.max(max, Math.abs(current - 127)), 0) /
    128
  );
};

const createMediaStream = (stream, isRecording, callback) => {
  // Create a new AudioContext
  const context = new AudioContext();

  // Create a media stream source node from the input stream
  const source = context.createMediaStreamSource(stream);

  // Create an analyzer node for audio analysis
  const analyzer = context.createAnalyser();

  // Connect the source node to the analyzer node
  source.connect(analyzer);

  // Function to continuously analyze audio data and invoke the callback
  const tick = () => {
    // Calculate the peak level using the getPeakLevel function
    const peak = getPeakLevel(analyzer);

    if (isRecording) {
      callback(peak);

      // Request the next animation frame for continuous analysis
      requestAnimationFrame(tick);
    }
  };

  // Start the continuous analysis loop
  tick();
};

export { createMediaStream };
You can learn more about the built-in requestAnimationFrame js method here.

Great now let’s use our helpers and speechToText api in useRecordVoice.

"use client";
import { useEffect, useState, useRef } from "react";
import { blobToBase64 } from "@/utils/blobToBase64";
import { createMediaStream } from "@/utils/createMediaStream";

export const useRecordVoice = () => {
  const [text, setText] = useState("");
  const [mediaRecorder, setMediaRecorder] = useState(null);
  const [recording, setRecording] = useState(false);
  const isRecording = useRef(false);
  const chunks = useRef([]);

  const startRecording = () => {
    if (mediaRecorder) {
      isRecording.current = true;
      mediaRecorder.start();
      setRecording(true);
    }
  };

  const stopRecording = () => {
    if (mediaRecorder) {
      isRecording.current = false;
      mediaRecorder.stop();
      setRecording(false);
    }
  };

  const getText = async (base64data) => {
    try {
      const response = await fetch("/api/speechToText", {
        method: "POST",
        headers: {
          "Content-Type": "application/json",
        },
        body: JSON.stringify({
          audio: base64data,
        }),
      }).then((res) => res.json());
      const { text } = response;
      setText(text);
    } catch (error) {
      console.log(error);
    }
  };

  const initialMediaRecorder = (stream) => {
    const mediaRecorder = new MediaRecorder(stream);

    mediaRecorder.onstart = () => {
      createMediaStream(stream)
      chunks.current = [];
    };

    mediaRecorder.ondataavailable = (ev) => {
      chunks.current.push(ev.data);
    };

    mediaRecorder.onstop = () => {
      const audioBlob = new Blob(chunks.current, { type: "audio/wav" });
      blobToBase64(audioBlob, getText);
    };

    setMediaRecorder(mediaRecorder);
  };

  useEffect(() => {
    if (typeof window !== "undefined") {
      navigator.mediaDevices
        .getUserMedia({ audio: true })
        .then(initialMediaRecorder);
    }
  }, []);

  return { recording, startRecording, stopRecording, text };
};
After that, go to our Microphone component and add the output of our text.

const Microphone = () => {
  const { startRecording, stopRecording, text } = useRecordVoice();

  return (
    <div className="flex flex-col justify-center items-center">
      <button
        onMouseDown={startRecording}
        onMouseUp={stopRecording}
        onTouchStart={startRecording}
        onTouchEnd={stopRecording}
        className="border-none bg-transparent w-10"
      >
        <IconMicrophone />
      </button>
      <p>{text}</p>
    </div>
  );
};

export { Microphone };
Save your changes and restart the server.

Next, go to the http://localhost:3000/ and do the following:

Press the microphone button
And say, “Hello world”
Result:


That’s it! You should now have a fully functional voice input in your Next.js project.

Link to GitHub where you can learn more about the structure and code here.
