<template>
	<div id="url" class="urlMessage">
		<div class="urlwrapper" v-if="urltype != 'empty' && !error">
			<metaMessage
				v-if="!loading"
				ref="metamessage"
				:type="urltype"
				:name="meta['og:site_name']"
				:title="meta['og:title']"
				:description="meta['og:description']"
				:image="previewImageUrl"
				:url="url"
				:h="meta['og:image:height']"
				:w="meta['og:image:width']"
				@updatedSize="updatedSize"
				@loaded="loaded"
			/>

			<div v-else>
				<div class="preloaderWrapperLocal">
					<linepreloader />
				</div>
			</div>
		</div>
	</div>
</template>

<style scoped lang="sass">
.preloaderWrapperLocal
    padding : $r 0
    text-align: center

.urlwrapper
    margin-top: 0.5 * $r
</style>

<script>
import { mapState } from "vuex";


var exts = ["bin", "dat", "swf", "doc", "docx", "sig", "tif", "cdr", "xls", "xlsx", "p7s", "mkv", "tmp", "db", "isz", "mdf", "jpg", "cr2", "fb2", "iso", "svg", "exe", "mdx", "vob", "ppt", "xls", "dcm", "vsd", "mov", "img", "pdf", "jpg", "jfif", "png"]

export default {
	name: "eventsurl",
	props: {
		url: String,
		preview: Boolean,
		data: Object,
	},
	components: {
		metaMessage: () => import("@/components/events/event/metaMessage/index.vue"),
	},
	data: function () {
		return {
			meta: {},
			loading: false,
			error: false,
			groups: {
				p: ["pocketnet.app", "bastyon.com"],
				pt: ["test.pocketnet.app", "test.bastyon.com"],
			},
		};
	},
	computed: mapState({
		auth: (state) => state.auth,
		clearurl: function () {
			var u = this.url || "";

			u = u.replace("http://", "https://");

			if (u.indexOf("https://") == -1) {
				u = "https://" + u;
			}

			return u;
		},
		previewImageUrl() {
			if (this.meta["og:image"])
				return this.core.mtrx.client.mxcUrlToHttp(this.meta["og:image"]);
		},
		urltype: function () {
			if (!this.url) return "empty";

			if (this.url.indexOf("embedVideo.php") > -1) {
				return "video";
			}

			var url = {};

			try {
				url = new URL(this.url);
			} catch (e) {
				this.error = e;
			}

			if (!url.pathname || url.pathname == "/") return "custom";

			var domain = window.pocketnetdomain || "pocketnet.app";

			if (this.url.indexOf("publicroom=") > -1) return "matrix";
			if (this.url.indexOf("connect=") > -1) return "matrix";

			if (this.url.indexOf("bastyon://") > -1) return "pocketnet";
			if (this.url.indexOf("pocketnet://") > -1) return "pocketnet";

			var m = _.find(this.groups, function (g) {
				return _.indexOf(g, url.host) > -1 && _.indexOf(g, domain) > -1;
			});

			if (m && this.url.indexOf("embedVideo.php") == -1 && this.url.indexOf("docs/") == -1 && this.url.indexOf("/blockexplorer") == -1) {
				return "pocketnet";
			}

			return "custom";
		},

		subtype : function(){
			if(this.urltype == 'custom'){
				if(this.clearurl && this.clearurl.indexOf && this.clearurl.indexOf('zoom.us') > -1){
					return 'zoom'
				}

				if(this.clearurl && this.clearurl.split){

					var ch = this.clearurl.split(/\//g)

					if (ch.length > 2){
						var ls = ch[ch.length - 1] || ''


						var lsc = ls.split('.')

						if (lsc.length == 2 && exts.indexOf(lsc[lsc.length - 1]) > -1){
							return 'file'
						}
					}

				}
			}
		}
	}),

	beforeMount: function () {
		if (this.urltype == "custom") {
			this.geturl();
		} else {
			this.loading = false;
		}
	},

	methods: {
		updatedSize: function (before) {
			this.$emit("updatedSize", before);
		},
		loaded: function (data) {
			this.$emit("loaded", data);
		},
		geturl: function () {
			//this.loading = true;

			if(this.subtype){
				if(this.subtype == 'zoom'){
					this.meta = {
						'og:title' : "Join our Cloud HD Video Meeting"
					}
				}

				if(this.subtype == 'file'){
					this.meta = {
						'og:title' : "File"
					}
				}

				return
			}

			this.core.mtrx.client
				.getUrlPreview(this.clearurl, 0)
				.then((response) => {

					var cl = {}

					_.each(response, (r, i) => {
						if(r) cl[i] = r
					})

					if(_.isEmpty(cl)){
						this.$emit('error', 'empty')
						return Promise.reject('empty')
					}

					this.meta = response;

				})
				.catch((error) => {
					this.meta = null;
					this.error = error

					this.$emit('error', error)
				}).finally(() => {
					this.loading = false;
				})
		},
	},
};
</script>
