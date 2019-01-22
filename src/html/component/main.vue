<style type="text/css">
hr {
	margin: 1rem 0;
}
#logs {
	height: 20rem;
	margin: 1rem 0;
	padding: 0.2rem 0.5rem;
	font-size: 90%;
	white-space: pre;
	background: #f1f1f1;
	border: 1px solid #e1e1e1;
	border-radius: 4px;
	overflow: auto;
}
</style>

<template>
	<form action id="settings">
		<div class="row">
			<div class="twelve columns">
				<h1 id="app-name">Recta Host</h1>
				<label>(PDAM TIRTA - Dinustek)</label>
				<label>Printer Adapter</label>
				<select class="u-full-width" v-model="printer.adapter">
					<option value="usb">USB</option>
					<option value="serial">Serial</option>
					<option value="network">LAN Network</option>
				</select>
			</div>
		</div>
		<!-- <component :is="printer.adapter" :data="printer.option" @change="optionChange"></component> -->
		<hr>
		<div class="row">
			<div class="twelve columns">
				<label>App Key</label>
				<input class="u-full-width" type="text" placeholder="Application Key" v-model="app.key">
			</div>
			<div class="twelve columns">
				<label>App Port</label>
				<input class="u-full-width" type="text" placeholder="Service Run on Port" v-model="app.port">
			</div>
			<div class="twelve columns">
				<label>
					<input type="checkbox" v-model="startup">
					<span class="label-body">Autostart on startup</span>
				</label>
				<label>
					<input type="checkbox" v-model="startup_on_stoped">
					<span class="label-body">Autostart on stoped</span>
				</label>
			</div>
		</div>
		<div class="row">
			<div class="twelve columns">
				<button class="button-default" @click.prevent="save">Save</button>
				<button class="button-default" @click.prevent="toggle">{{ isStart ? 'Stop' : 'Start' }}</button>
				<button class="button-default" @click.prevent="test" v-show="isStart">Test Print</button>
			</div>
		</div>
		<div class="row">
			<div class="twelve columns">
				<label>
					Log
					<a href="#" @click.prevent="clearLog">[ Clear ]</a>
				</label>
				<div class="u-full-width" id="logs">{{ logs.join('\n') }}</div>
			</div>
		</div>
	</form>
</template>

<script type="text/javascript">
import _ from "lodash";
import { ipcRenderer } from "electron";
import Service from "../../js/service.js";
import { AUTOLUNCHER, CFG, PRINT_TEST } from "../../js/constant.js";
import Usb from "./setting/usb.vue";
import Serial from "./setting/serial.vue";
import Network from "./setting/network.vue";

export default {
	components: { Usb, Serial, Network },
	data() {
		return {
			printer: {
				adapter: "usb",
				option: {}
			},
			app: {
				port: 1811,
				key: ""
			},
			startup: true,
			startup_on_stoped: true,
			restart_attempt: 0,
			service: null,
			isStart: false,
			logs: []
		};
	},
	created() {
		_.assign(this.$data, CFG.get());

		ipcRenderer.on("start", () => {
			this.start();
		});
		ipcRenderer.on("stop", () => {
			this.stop();
		});
	},
	mounted() {
		AUTOLUNCHER.isEnabled().then(isEnabled => {
			this.startup = isEnabled;

			if (isEnabled) this.start();
		});
	},
	watch: {
		isStart(val) {
			if (val) {
				ipcRenderer.send("started");
			} else {
				ipcRenderer.send("stoped");
				if (this.startup_on_stoped && this.restart_attempt <= 100) {
					this.start(true);
					this.restart_attempt++;
				}
			}
		}
	},
	methods: {
		save() {
			const promise = this.startup
				? AUTOLUNCHER.enable()
				: AUTOLUNCHER.disable();

			promise
				.then(() => {
					CFG.set("printer", this.printer);
					CFG.set("app", this.app);

					this.log("Config Saved");
				})
				.catch(e => {
					this.log(e);
				});
		},
		toggle() {
			!this.isStart ? this.start() : this.stop();
		},
		start(restarted = false) {
			try {
				this.service = new Service();
				this.service.on("open", () => {
					this.isStart = true;
					this.restart_attempt = 0;

					if (restarted) {
						this.log("Service Restarted");
					} else {
						this.log("Service Started");
					}
				});
				this.service.on("close", () => {
					this.isStart = false;
					this.service = null;

					this.log("Service Stoped");
				});
				this.service.on("message", () => {
					this.log("Receive printing message");
				});
				this.service.on("error", e => {
					this.log(e);
				});
				this.service.start().catch(e => {
					this.log(e);
				});
			} catch (e) {
				this.log(e);
			}
		},
		stop() {
			try {
				this.service.stop();
			} catch (e) {
				this.log(e);
			}
		},
		test() {
			try {
				this.service.printer.print(PRINT_TEST);
			} catch (e) {
				this.log(e);
			}
		},
		optionChange(option) {
			_.assign(this.printer.option, option);
		},
		log(text) {
			const date = new Date();

			this.logs.unshift(`${date.toLocaleString()} : ${text} !`);
		},
		clearLog() {
			this.logs = [];
		}
	}
};
</script>
