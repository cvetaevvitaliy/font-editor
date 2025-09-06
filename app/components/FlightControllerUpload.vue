<script setup lang="ts">
import { useFontStore } from "../../stores/fontStore"

// Type definitions for Web Serial API
interface SerialPort {
	open(options: { baudRate: number }): Promise<void>
	close(): Promise<void>
	readonly readable: ReadableStream<Uint8Array>
	readonly writable: WritableStream<Uint8Array>
	getInfo(): SerialPortInfo
}

interface SerialPortInfo {
	usbVendorId?: number
	usbProductId?: number
}

interface Serial {
	getPorts(): Promise<SerialPort[]>
	requestPort(): Promise<SerialPort>
}

declare global {
	interface Navigator {
		serial: Serial
	}
}

const fontStore = useFontStore()

const selectedPort = ref<SerialPort | null>(null)
const availablePorts = ref<SerialPort[]>([])
const isUploading = ref(false)
const uploadStatus = ref<string | null>(null)

async function refreshPorts() {
	try {
		const ports = await navigator.serial.getPorts()
		availablePorts.value = ports
		if (ports.length > 0 && ports[0]) {
			selectedPort.value = ports[0] as SerialPort
		}
	} catch {
		availablePorts.value = []
		selectedPort.value = null
	}
}

async function requestPort() {
	try {
		const port = await navigator.serial.requestPort()
		availablePorts.value.push(port as SerialPort)
		selectedPort.value = port as SerialPort
	} catch (err) {
		console.warn("Serial port selection cancelled or failed", err)
	}
}

onMounted(() => {
	// Check if Web Serial API is available
	if ("serial" in navigator) {
		refreshPorts()
	} else {
		uploadStatus.value = "Web Serial API not supported in this browser"
	}
})

async function uploadToDevice() {
	if (!selectedPort.value) {
		uploadStatus.value = "Please select a serial port first."
		return
	}
	if (!fontStore.hasData) {
		uploadStatus.value = "No font data to upload."
		return
	}

	isUploading.value = true
	uploadStatus.value = "Uploading..."

	try {
		// Request backend for ready MSP packets (base64)
		const response = await fetch("/api/json-to-msp-packets", {
			method: "POST",
			headers: { "Content-Type": "application/json" },
			body: JSON.stringify({
				fontData: {
					metadata: fontStore.metadata || "MAX7456",
					characterCount: fontStore.characterCount || 256,
					characters: fontStore.characters
				}
			})
		})

		if (!response.ok) {
			throw new Error(
				`Server returned ${response.status}: ${response.statusText}`
			)
		}

		const result = await response.json()
		if (!result.success) {
			throw new Error(result.error || "Unknown error")
		}

		await selectedPort.value.open({ baudRate: 115200 })

		const writer = selectedPort.value.writable.getWriter()

		const total = result.packets.length
		for (let i = 0; i < total; i++) {
			uploadStatus.value = `${i + 1}/${total}`
			const packet = Uint8Array.from(atob(result.packets[i]), (c) =>
				c.charCodeAt(0)
			)
			await writer.write(packet)
			await new Promise((r) => setTimeout(r, 60))
		}

		writer.releaseLock()
		await selectedPort.value.close()

		uploadStatus.value = "Upload successful!"
	} catch (error) {
		uploadStatus.value =
			"Upload failed: " +
			(error instanceof Error ? error.message : String(error))
	} finally {
		isUploading.value = false
	}
}
</script>

<template>
	<div class="flex flex-col gap-4">
		<div class="flex flex-col gap-4">
			<h3 class="text-lg font-semibold text-primary-400">
				Flight Controller Upload
			</h3>

			<div class="flex flex-col gap-2 text-text text-sm max-w-[50ch]">
				<p>
					Upload your font directly to a flight controller via serial
					connection. This feature uses the Web Serial API to communicate with
					your device.
				</p>
				<p>
					Click "Open Serial Port" to select your flight controller's serial
					port, then click "Upload to Device" to transfer the font data.
				</p>
			</div>

			<div class="flex gap-4 items-center">
				<!-- Open Serial Port Button -->
				<UButton
					color="primary"
					variant="soft"
					icon="i-lucide-plug"
					@click="requestPort"
				>
					Open Serial Port
				</UButton>

				<!-- Upload button -->
				<UButton
					:loading="isUploading"
					:disabled="!selectedPort || !fontStore.hasData"
					color="secondary"
					icon="i-heroicons-arrow-up"
					variant="soft"
					@click="uploadToDevice"
				>
					Upload to Device
				</UButton>
			</div>

			<div v-if="uploadStatus" class="mt-2">
				<p
					class="text-sm leading-tight"
					:class="{
						'text-green-500': uploadStatus.includes('successful'),
						'text-red-500': uploadStatus.includes('failed'),
						'text-blue-500':
							!uploadStatus.includes('successful') &&
							!uploadStatus.includes('failed')
					}"
				>
					{{ uploadStatus }}
				</p>
			</div>

			<div v-if="selectedPort" class="mt-2">
				<p class="text-xs text-neutral-400">
					Connected to:
					{{ selectedPort.getInfo().usbProductId || "Serial Port" }}
				</p>
			</div>
		</div>
	</div>
</template>
