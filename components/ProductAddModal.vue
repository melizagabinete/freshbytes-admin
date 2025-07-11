<script setup>
    import { ref, watch } from 'vue'

    const props = defineProps({
        showAddProductModal: {
            type: Boolean,
            required: true
        },
        newProduct: {
            type: Object,
            required: true
        },
        sellers: {
            type: Array,
            required: true
        },
        subcategories: {
            type: Array,
            required: true
        },
        users: {
            type: Array,
            required: true
        },
        categories: {
            type: Array,
            required: true
        }
    })

    // grab the emit helper
    const emit = defineEmits(['closeAddProductModal','addProduct'])

    // wrap the emit so the template can call it
    function onAdd() {
        emit('addProduct')
        emit('update:productImages', filesArray)
    }

    function closeAddProductModal() {
        emit('closeAddProductModal')
    }

    let debounceTimeout = null

    const locationQuery = ref('')
    const locationSuggestions = ref([])

    watch(locationQuery, (query) => {
        clearTimeout(debounceTimeout)
        if (!query) {
            locationSuggestions.value = []
            return
        }
        debounceTimeout = setTimeout(async () => {
            const url = `https://nominatim.openstreetmap.org/search?format=json&q=${encodeURIComponent(query)}&limit=5`
            const res = await fetch(url, { headers: { 'Accept-Language': 'en' } })
            locationSuggestions.value = await res.json()
        }, 400) // 400ms debounce
    })

    function selectLocation(suggestion) {
        props.newProduct.product_location = suggestion.display_name
        locationQuery.value = suggestion.display_name
        locationSuggestions.value = []
    }

    // Keep input in sync with newProduct
    watch(() => props.newProduct.product_location, (val) => {
        if (val !== locationQuery.value) locationQuery.value = val || ''
    })

    // In ProductAddModal.vue
    watch(() => props.showAddProductModal, (val) => {
    if (!val) {
        productImagesPreview.value = [];
    }
    });

    function formatDate(dateStr) {
        if (!dateStr) return 'N/A';
        const date = new Date(dateStr);
        if (isNaN(date)) return dateStr; // fallback if invalid
        return date.toLocaleDateString('en-US', {
            year: 'numeric',
            month: 'long',
            day: 'numeric',
            hour: '2-digit',
            minute: '2-digit'
        });
    }

    // Add this near your other refs
    const productImages = ref([]) // Array of File objects
    const productImagesPreview = ref([]) // Array of preview URLs

    function onProductImagesChange(e) {
        const files = Array.from(e.target.files);
        // Add new files to the existing ones, avoiding duplicates by name+size
        const existing = productImages.value.map(f => f.name + f.size);
        files.forEach(file => {
            if (!existing.includes(file.name + file.size)) {
            productImages.value.push(file);
            const reader = new FileReader();
            reader.onload = ev => productImagesPreview.value.push(ev.target.result);
            reader.readAsDataURL(file);
            }
        });
        // Reset input value so the same file can be selected again if needed
        e.target.value = '';
    }

    function removeImage(idx) {
        productImages.value.splice(idx, 1);
        productImagesPreview.value.splice(idx, 1);
    }

    function onProductImagesDrop(e) {
        const files = Array.from(e.dataTransfer.files).filter(f => f.type.startsWith('image/'));
        const existing = productImages.value.map(f => f.name + f.size);
        files.forEach(file => {
            if (!existing.includes(file.name + file.size)) {
            productImages.value.push(file);
            const reader = new FileReader();
            reader.onload = ev => productImagesPreview.value.push(ev.target.result);
            reader.readAsDataURL(file);
            }
        });
    }
</script>

<template>
    <div v-if="showAddProductModal" class="fixed inset-0 z-50 flex items-center justify-center bg-gray-800/30">
        <div class="bg-white text-gray-900 rounded-lg shadow-lg w-full max-w-2xl p-8 relative h-screen overflow-y-auto">
            <button class="absolute top-4 right-4 text-gray-400 hover:text-gray-700 text-2xl"
                @click="closeAddProductModal">&times;</button>
            <h2 class="text-2xl font-semibold mb-6">Add Product</h2>
            <form class="space-y-4">
                <div class="grid grid-cols-2 gap-4">
                    <div>
                        <label class="block mb-1 font-medium">Seller Name</label>
                        <select class="w-full px-3 py-2 rounded bg-gray-100 border border-gray-300 focus:outline-none"
                            v-model="newProduct.seller_id" required>
                            <option disabled value="">Select seller</option>
                            <option v-for="seller in sellers" :key="seller.seller_id" :value="seller.seller_id">
                                {{users.find(u => u.user_id === seller.user_id)?.user_name || 'Unknown'}}
                            </option>
                        </select>
                    </div>
                    <div>
                        <label class="block mb-1 font-medium">Product Name</label>
                        <input type="text"
                            class="w-full px-3 py-2 rounded bg-gray-100 border border-gray-300 focus:outline-none"
                            placeholder="Type product name" v-model="newProduct.product_name" />
                    </div>
                    <div>
                        <label class="block mb-1 font-medium">Sub-Category</label>
                        <select class="w-full px-3 py-2 rounded bg-gray-100 border border-gray-300 focus:outline-none"
                            v-model="newProduct.sub_category_id">
                            <option disabled value="">Select sub-category</option>
                            <option v-for="sub in subcategories" :key="sub.sub_category_id"
                                :value="sub.sub_category_id">
                                {{ sub.sub_category_name }}
                            </option>
                        </select>
                    </div>
                    <div>
                        <label class="block mb-1 font-medium">Price</label>
                        <input type="number"
                            class="w-full px-3 py-2 rounded bg-gray-100 border border-gray-300 focus:outline-none"
                            placeholder="₱0.00" v-model="newProduct.product_price" />
                    </div>
                    <div>
                        <label class="block mb-1 font-medium">Discounted Price</label>
                        <input type="number"
                            class="w-full px-3 py-2 rounded bg-gray-100 border border-gray-300 focus:outline-none"
                            placeholder="₱0.00" v-model="newProduct.product_discountedPrice" />
                    </div>
                    <div>
                        <label class="block mb-1 font-medium">Product Status</label>
                        <select class="w-full px-3 py-2 rounded bg-gray-100 border border-gray-300 focus:outline-none"
                            v-model="newProduct.product_status" required>
                            <option disabled value="">Select status</option>
                            <option value="FRESH">Fresh</option>
                            <option value="SLIGHTLY_WITHERED">Slightly Withered</option>
                            <option value="ROTTEN">Rotten</option>
                        </select>
                    </div>
                    <div>
                        <label class="block mb-1 font-medium">Product Location</label>
                        <div class="relative">
                            <input
                                type="text"
                                class="w-full px-3 py-2 rounded bg-gray-100 border border-gray-300 focus:outline-none"
                                placeholder="Type your Product Location"
                                v-model="locationQuery"
                                autocomplete="off"
                            />
                            <ul v-if="locationSuggestions.length"
                                class="absolute bg-white border border-gray-300 rounded shadow mt-1 z-50 w-full max-h-48 overflow-auto">
                                <li v-for="suggestion in locationSuggestions"
                                    :key="suggestion.place_id || suggestion.osm_id"
                                    @click="selectLocation(suggestion)"
                                    class="px-3 py-2 hover:bg-gray-100 cursor-pointer"
                                >
                                    {{ suggestion.display_name }}
                                </li>
                            </ul>
                        </div>
                    </div>
                    <div>
                        <label class="block mb-1 font-medium">Quantity</label>
                        <input type="number"
                            class="w-full px-3 py-2 rounded bg-gray-100 border border-gray-300 focus:outline-none"
                            placeholder="0" v-model="newProduct.quantity" />
                    </div>
                    <div>
                        <label class="block mb-1 font-medium">Posted Date & Time</label>
                        <input type="datetime-local"
                            class="w-full px-3 py-2 rounded bg-gray-100 border border-gray-300 focus:outline-none"
                            v-model="newProduct.post_date" />
                    </div>
                    <div>
                        <label class="block mb-1 font-medium">Harvest Date & Time</label>
                        <input type="datetime-local"
                            class="w-full px-3 py-2 rounded bg-gray-100 border border-gray-300 focus:outline-none"
                            v-model="newProduct.harvest_date" />
                    </div>
                </div>
                <div>
                    <label class="block mb-1 font-medium">Brief Description</label>
                    <textarea class="w-full px-3 py-2 rounded bg-gray-100 border border-gray-300 focus:outline-none"
                        rows="3" placeholder="Write product description here"
                        v-model="newProduct.product_brief_description"></textarea>
                </div>
                <div>
                    <label class="block mb-1 font-medium">Detailed Description</label>
                    <textarea class="w-full px-3 py-2 rounded bg-gray-100 border border-gray-300 focus:outline-none"
                        rows="3" placeholder="Write product description here"
                        v-model="newProduct.product_full_description"></textarea>
                </div>
                <div>
                    <label class="block mb-1 font-medium">Product Images</label>
                    <div
                        :class="[
                        'w-full h-40 border-2 border-dashed border-gray-300 rounded flex flex-col items-center justify-center bg-gray-50 cursor-pointer transition relative',
                        productImagesPreview.length ? 'overflow-x-auto' : 'text-gray-400'
                        ]"
                        @click="$refs.productImagesInput.click()"
                        @dragover.prevent
                        @drop.prevent="onProductImagesDrop"
                    >
                        <template v-if="!productImagesPreview.length">
                        <svg class="w-12 h-12 mb-2" fill="none" stroke="currentColor" stroke-width="2" viewBox="0 0 24 24">
                            <path d="M12 4v16m8-8H4" />
                        </svg>
                        <span class="font-semibold">Click to upload</span> or drag and drop
                        <div class="text-xs mt-1">SVG, PNG, JPG or GIF (MAX. 800×400px)</div>
                        </template>
                        <template v-else>
                        <div class="flex flex-wrap gap-2 justify-center items-center w-full h-full">
                            <div
                            v-for="(img, idx) in productImagesPreview"
                            :key="idx"
                            class="relative w-20 h-20 rounded overflow-hidden border bg-white flex items-center justify-center"
                            >
                            <img :src="img" class="object-cover w-full h-full" />
                            <button
                                type="button"
                                class="absolute top-0 right-0 bg-white bg-opacity-80 hover:bg-red-500 hover:text-white text-gray-700 rounded-full w-6 h-6 flex items-center justify-center text-xs"
                                @click.stop="removeImage(idx)"
                                title="Remove image"
                            >
                                &times;
                            </button>
                            </div>
                        </div>
                        <span class="absolute bottom-2 left-1/2 -translate-x-1/2 text-xs text-gray-500">
                            Images uploaded ({{ productImagesPreview.length }})
                        </span>
                        </template>
                        <input
                        ref="productImagesInput"
                        type="file"
                        accept="image/*"
                        multiple
                        class="hidden"
                        @change="onProductImagesChange"
                        />
                    </div>
                </div>
                <div class="flex justify-end space-x-2 mt-6">
                    <button type="button" class="bg-green-600 hover:bg-green-700 text-white px-4 py-2 rounded"
                        @click="onAdd">Add
                        product</button>
                    <button type="button" class="bg-gray-200 hover:bg-gray-300 text-gray-700 px-4 py-2 rounded"
                        @click="closeAddProductModal">Cancel</button>
                </div>
            </form>
        </div>
    </div>
</template>