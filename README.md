import React, { useState, useEffect, useCallback } from 'react';
import { LineChart, Line, XAxis, YAxis } from 'recharts';

const Dashboard = () => {
  const [voltajeAC, setVoltajeAC] = useState(0);
  const [voltajeDC, setVoltajeDC] = useState(0);
  const [amperaje, setAmperaje] = useState(0);
  const [mamper, setMamper] = useState(0);
  const [ldrValues, setLdrValues] = useState([
    { name: 'LDR 1', value: 0 }, 
    { name: 'LDR 2', value: 0 }, 
    { name: 'LDR 3', value: 0 }, 
    { name: 'LDR 4', value: 0 }
  ]);
  const [ws, setWs] = useState(null);
  const [joystickValues, setJoystickValues] = useState([
    { x: 0, y: 0 }, 
    { x: 0, y: 0 }, 
    { x: 0, y: 0 }, 
    { x: 0, y: 0 }
  ]);

  // Conectar WebSocket
  useEffect(() => {
    const websocket = new WebSocket('ws://esp32-ip-address:81');
    
    websocket.onopen = () => {
      console.log('Conectado al ESP32');
    };

    websocket.onmessage = (event) => {
      const data = JSON.parse(event.data);
      setVoltajeAC(data.voltajeAC);
      setVoltajeDC(data.voltajeDC);
      setAmperaje(data.amperaje);
      setMamper(data.mamper);
      setLdrValues(data.ldrValues);
    };

    setWs(websocket);

    return () => {
      websocket.close();
    };
  }, []);

  // Función para manejar el movimiento del joystick
  const handleJoystickMove = useCallback((index, x, y) => {
    const newValues = [...joystickValues];
    newValues[index] = { x, y };
    setJoystickValues(newValues);
    
    if (ws && ws.readyState === WebSocket.OPEN) {
      ws.send(JSON.stringify({
        type: 'joystick',
        index: index,
        x: x,
        y: y
      }));
    }
  }, [ws, joystickValues]);

  const Joystick = ({ index }) => {
    const [isDragging, setIsDragging] = useState(false);
    const [position, setPosition] = useState({ x: 0, y: 0 });
    
    const handleMouseDown = (e) => {
      setIsDragging(true);
    };

    const handleMouseMove = (e) => {
      if (isDragging) {
        const rect = e.target.getBoundingClientRect();
        const x = Math.min(Math.max((e.clientX - rect.left) / rect.width * 200 - 100, -100), 100);
        const y = Math.min(Math.max((e.clientY - rect.top) / rect.height * 200 - 100, -100), 100);
        setPosition({ x, y });
        handleJoystickMove(index, x, y);
      }
    };

    const handleMouseUp = () => {
      setIsDragging(false);
      setPosition({ x: 0, y: 0 });
      handleJoystickMove(index, 0, 0);
    };

    return (
      <div 
        className="w-32 h-32 bg-gray-200 rounded-full relative cursor-pointer"
        onMouseDown={handleMouseDown}
        onMouseMove={handleMouseMove}
        onMouseUp={handleMouseUp}
        onMouseLeave={handleMouseUp}
      >
        <div 
          className="w-12 h-12 bg-blue-500 rounded-full absolute"
          style={{
            left: `${50 + (position.x / 200 * 100)}%`,
            top: `${50 + (position.y / 200 * 100)}%`,
            transform: 'translate(-50%, -50%)'
          }}
        />
      </div>
    );
  };

  return (
    <div className="container mx-auto p-4 pt-6">
      <div className="flex flex-wrap justify-center mb-4">
        <div className="bg-gray-200 border-2 border-dashed rounded-xl w-16 h-16 mr-4">
          <p className="text-center mt-6">ESP32</p>
        </div>
        {[1, 2, 3, 4].map((num) => (
          <div key={num} className="bg-gray-200 border-2 border-dashed rounded-xl w-16 h-16 mr-4">
            <p className="text-center mt-6">LDR {num}</p>
          </div>
        ))}
      </div>

      <div className="flex flex-wrap justify-center mb-4">
        <div className="bg-white rounded-lg shadow-md p-4 w-1/2 xl:w-1/4 mr-4">
          <p className="text-lg font-bold mb-2">Voltaje AC</p>
          <p className="text-3xl font-bold">{voltajeAC.toFixed(2)}V</p>
        </div>
        <div className="bg-white rounded-lg shadow-md p-4 w-1/2 xl:w-1/4 mr-4">
          <p className="text-lg font-bold mb-2">Voltaje DC</p>
          <p className="text-3xl font-bold">{voltajeDC.toFixed(2)}V</p>
        </div>
        <div className="bg-white rounded-lg shadow-md p-4 w-1/2 xl:w-1/4 mr-4">
          <p className="text-lg font-bold mb-2">Amperaje</p>
          <p className="text-3xl font-bold">{amperaje.toFixed(2)}A</p>
        </div>
        <div className="bg-white rounded-lg shadow-md p-4 w-1/2 xl:w-1/4">
          <p className="text-lg font-bold mb-2">Mamper</p>
          <p className="text-3xl font-bold">{mamper.toFixed(2)}mA</p>
        </div>
      </div>

      <div className="flex flex-wrap justify-center mb-4">
        {[0, 1, 2, 3].map((index) => (
          <div key={index} className="m-4">
            <p className="text-center mb-2">Joystick {index + 1}</p>
            <Joystick index={index} />
          </div>
        ))}
      </div>

      <div className="bg-white rounded-lg shadow-md p-4 w-full">
        <p className="text-lg font-bold mb-2">Comportamiento de LDR</p>
        <LineChart width={800} height={400} data={ldrValues}>
          <XAxis dataKey="name" />
          <YAxis />
          <Line type="monotone" dataKey="value" stroke="#8884d8" />
        </LineChart>
      </div>
    </div>
  );
};

export default Dashboard;

