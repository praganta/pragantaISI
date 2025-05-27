#INTERKONEKSI
==============
====MODBUS====
==============
```
->> CARGO.TOML MODBUS <<-
[package]
name = "sht20"
version = "0.1.0"
edition = "2021"


[dependencies]
tokio = { version = "1", features = ["full"] }
tokio-modbus = "0.5"
tokio-serial = "5.4"
serde = { version = "1", features = ["derive"] }
serde_json = "1.0"
chrono = "0.4"




->> MAIN.RS MODBUS <<-
use tokio_modbus::prelude::*;
use tokio_serial::SerialStream;
use std::time::Duration;
use tokio::net::TcpStream;
use tokio::io::AsyncWriteExt;
use chrono::Utc;
use serde::Serialize;

#[derive(Serialize)]
struct SensorPayload {
    timestamp: String,
    sensor_id: String,
    location: String,
    process_stage: String,
    temperature_celsius: f32,
    humidity_percent: f32,
}

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let builder = tokio_serial::new("/dev/ttyUSB0", 9600)
        .timeout(Duration::from_secs(1))
        .parity(tokio_serial::Parity::None)
        .stop_bits(tokio_serial::StopBits::One)
        .data_bits(tokio_serial::DataBits::Eight)
        .flow_control(tokio_serial::FlowControl::None);

    let port = SerialStream::open(&builder)?;
    let mut ctx = rtu::connect_slave(port, Slave(0x01)).await?;

    // Koneksi ke TCP server
    let mut stream = TcpStream::connect("127.0.0.1:7878").await?;
    println!("Connected to TCP server.");

    loop {
        let response = ctx.read_input_registers(0x0001, 2).await?;
        let raw_temp = response[0];
        let raw_humi = response[1];

        let temperature = raw_temp as f32 / 10.0;
        let humidity = raw_humi as f32 / 10.0;

        println!("Temperature: {} °C | Humidity: {} %", temperature, humidity);

        let payload = SensorPayload {
            timestamp: Utc::now().to_rfc3339(),
            sensor_id: "SHT20-PascaPanen-001".to_string(),
            location: "Gudang Fermentasi 1".to_string(),
            process_stage: "Fermentasi".to_string(),
            temperature_celsius: temperature,
            humidity_percent: humidity,
        };

        let json_payload = serde_json::to_string(&payload)?;
        stream.write_all(json_payload.as_bytes()).await?;
        stream.write_all(b"\n").await?; // Penting! agar TCP server bisa baca per baris

        tokio::time::sleep(Duration::from_secs(5)).await;
    }
}

#TCP SERVER
================
===TCP SERVER===
================

->> CARGO.TOML TCP SERVER <<-
<[package]
name = "tcp_server"
version = "0.1.0"
edition = "2021"

[dependencies]
tokio = { version = "1", features = ["full"] }
serde = { version = "1.0", features = ["derive"] }
serde_json = "1.0"
influxdb2 = "0.4.5"
futures = "0.3"  # Untuk stream::once
chrono = "0.4"  # Untuk parsing timestamp





->> MAIN.RS TCP SERVER <<-
use chrono::DateTime;
use futures::stream;
use influxdb2::{models::DataPoint, Client};
use serde::Deserialize;
use std::error::Error;
use tokio::io::{AsyncBufReadExt, BufReader};
use tokio::net::TcpListener;

#[derive(Debug, Deserialize)]
struct SensorData {
    timestamp: String,
    sensor_id: String,
    location: String,
    process_stage: String,
    temperature_celsius: f64,
    humidity_percent: f64,
}

#[tokio::main]
async fn main() -> Result<(), Box<dyn Error>> {
    // KONFIGURASI INFLUX (TOKEN,ORG,BUCKET)
    let token = "CdfAF6AlZ1a_XZnS_DScpSf68WYgIDYpKj-tKMcbvrYhs0uHrT6Mn0njqGTh88W0M8n8BJhERAxAQwIMJrs4jA==";
    let org = "myorg";
    let bucket = "fermentation";

    let client = Client::new("http://localhost:8086", org, token);

    let listener = TcpListener::bind("127.0.0.1:7878").await?;
    println!("Server listening on port 7878");

    loop {
        let (socket, _) = listener.accept().await?;
        let client = client.clone();
        let bucket = bucket.to_string();
        tokio::spawn(async move {
            let reader = BufReader::new(socket);
            let mut lines = reader.lines();

            while let Ok(Some(line)) = lines.next_line().await {
                match serde_json::from_str::<SensorData>(&line) {
                    Ok(data) => {
                        println!("Received: {:?}", data);

                        let parsed_time: DateTime<chrono::Utc> =
                            match DateTime::parse_from_rfc3339(&data.timestamp) {
                                Ok(t) => t.with_timezone(&chrono::Utc),
                                Err(e) => {
                                    eprintln!("Invalid timestamp: {}", e);
                                    continue;
                                }
                            };

                        let point = DataPoint::builder("environtment")
                            .tag("sensor_id", data.sensor_id)
                            .tag("location", data.location)
                            .tag("process_stage", data.process_stage)
                            .field("temperature", data.temperature_celsius)
                            .field("humidity", data.humidity_percent)
                            .timestamp(parsed_time.timestamp_nanos_opt().unwrap_or(0))
                            .build()
                            .unwrap();

                        if let Err(e) = client
                            .write(&bucket, stream::once(async { point }))
                            .await
                        {
                            eprintln!("Write to InfluxDB failed: {}", e);
                        }
                    }
                    Err(e) => eprintln!("Invalid JSON received: {}", e),
                }
            }
        });
    }
}
