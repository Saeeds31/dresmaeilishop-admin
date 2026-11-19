<template>
    <div class="qpc-wrap" v-if="checkPermission(['product_store'])">
        <!-- ============ ستون راست: ادیتور ============ -->
        <main class="qpc-editor">
            <div class="editor-header">
                <h3><i class="bi bi-magic"></i> ساخت سریع محصول</h3>
                <span class="hint">فقط تایپ کن، Enter بزن، برو خط بعد 😎</span>
            </div>

            <div class="smart-input">
                <!-- خطوط commit شده -->
                <div v-for="(line, i) in committedLines" :key="'c' + i" class="line-committed"
                    :class="'line-' + line.type">
                    <span class="line-icon"><i :class="lineIcon(line.type)"></i></span>
                    <span class="line-content">
                        <template v-if="line.type === 'category'">
                            <span class="cat-box">{{ line.value }}</span>
                        </template>
                        <template v-else-if="line.type === 'price'">
                            {{ fmt(line.raw.base) }} تومان
                            <template v-if="line.raw.final !== line.raw.base">
                                → <b>{{ fmt(line.raw.final) }}</b>
                                <span v-if="line.raw.percent" class="mini-badge">{{ line.raw.percent }}%</span>
                                <span v-if="line.raw.endDate" class="mini-badge date">{{ line.raw.endDate }}</span>
                            </template>
                        </template>
                        <template v-else-if="line.type === 'stock'">
                            موجودی: {{ fmt(line.value) }}
                        </template>
                        <template v-else-if="line.type === 'title'">
                            <b>{{ line.value }}</b>
                        </template>
                        <template v-else>
                            {{ truncate(line.value, 200) }}
                        </template>
                    </span>
                </div>

                <!-- خط جاری -->
                <div class="line-current" :class="'line-' + currentLineType">
                    <span class="line-icon"><i :class="lineIcon(currentLineType)"></i></span>
                    <input ref="inputRef" v-model="currentInput" class="smart-line-input"
                        :placeholder="currentPlaceholder" enterkeyhint="done" autocomplete="off" autocorrect="off"
                        autocapitalize="off" spellcheck="false" @keydown.enter.prevent.stop="safeCommit"
                        @keyup.enter.prevent.stop="safeCommit" @keydown.tab.prevent="handleTab"
                        @keydown.backspace="handleBackspace" @beforeinput="onBeforeInput" @blur="onBlurInput" />

                    <div v-if="currentLineType === 'category' && suggestions.length" class="suggestions">
                        <button v-for="s in suggestions" :key="s.id" class="suggestion-chip" type="button"
                            @mousedown.prevent @click="pickCategory(s)">
                            <i class="bi bi-tag"></i> {{ s.label }}
                        </button>
                    </div>
                </div>

                <div v-if="parsed.categories.length" class="selected-cats">
                    <span class="sc-label">دسته‌ها:</span>
                    <span v-for="c in parsed.categories" :key="c.id" class="sc-chip">
                        {{ c.label }}
                        <i class="bi bi-x" @click="removeCategory(c.id)"></i>
                    </span>
                </div>
            </div>

            <!-- تصاویر -->
            <div class="upload-section">
                <h4><i class="bi bi-images"></i> تصاویر محصول</h4>
                <div class="upload-box" @click="$refs.fileInput.click()" @dragover.prevent @drop.prevent="onDrop">
                    <input ref="fileInput" type="file" multiple accept="image/*" hidden @change="onFiles" />
                    <i class="bi bi-cloud-arrow-up"></i>
                    <p>کلیک یا Drag & Drop کنید</p>
                </div>
                <div v-if="images.length" class="thumbs">
                    <div v-for="(img, i) in images" :key="i" class="thumb" :class="{ main: i === 0 }">
                        <img :src="img.url" />
                        <span v-if="i === 0" class="main-tag">اصلی</span>
                        <i class="bi bi-x remove-thumb" @click="removeImage(i)"></i>
                    </div>
                </div>
            </div>

            <!-- ویژگی‌ها -->
            <div class="variant-section">
                <h4><i class="bi bi-palette"></i> ویژگی‌های محصول</h4>

                <div class="vi-group full">
                    <label>ویژگی‌ها</label>
                    <Treeselect v-model="selectedAttributes" :multiple="true" :options="attributes"
                        :normalizer="attributeNormalizer" placeholder="مثلاً رنگ، سایز، جنس..." />
                </div>

                <template v-for="attributeId in selectedAttributes" :key="attributeId">
                    <div class="vi-group full">
                        <label>انتخاب {{ attrName(attributeId) }}:</label>
                        <Treeselect :valueFormat="'object'" v-model="attributeValue[attributeId]" :multiple="true"
                            :options="attributeOptionsFor(attributeId)" :normalizer="attributeValuesNormalizer"
                            :placeholder="`انتخاب ${attrName(attributeId)}...`" />
                    </div>
                </template>

                <p class="variant-note">
                    <i class="bi bi-info-circle"></i>
                    قیمت و موجودی همه تنوع‌ها از مقدار اصلی محصول گرفته می‌شود؛ جدولی ساخته نمی‌شود.
                </p>
            </div>

            <!-- ذخیره -->
            <div class="save-bar">
                <button class="btn-save" :disabled="!canSave || loading" @click="save">
                    <span v-if="loading" class="spinner"></span>
                    <i v-else class="bi bi-cloud-check"></i>
                    <span>{{ loading ? 'در حال ذخیره...' : 'ذخیره محصول' }}</span>
                </button>
                <button class="btn-reset" :disabled="loading" @click="resetAll">
                    <i class="bi bi-arrow-counterclockwise"></i> پاک‌کردن
                </button>
            </div>

            <div v-if="errors.length" class="errors-box">
                <h5><i class="bi bi-exclamation-triangle"></i> خطاها</h5>
                <ul>
                    <li v-for="(e, i) in errors" :key="i">{{ e }}</li>
                </ul>
            </div>
        </main>
        <!-- ============ ستون چپ: پیش‌نمایش ============ -->
        <aside class="qpc-preview">
            <div class="preview-header">
                <i class="bi bi-eye"></i>
                <span>پیش‌نمایش زنده</span>
                <span v-if="parsed.title" class="badge-live">Live</span>
            </div>

            <div class="product-card" :class="{ empty: !parsed.title }">
                <div class="card-image">
                    <img v-if="previewImage" :src="previewImage" alt="preview" />
                    <div v-else class="img-placeholder">
                        <i class="bi bi-image"></i>
                        <span>تصویر اصلی</span>
                    </div>
                    <span v-if="discountLabel" class="discount-badge">{{ discountLabel }}</span>
                </div>

                <div class="card-body">
                    <h2 class="product-title">{{ parsed.title || 'عنوان محصول...' }}</h2>

                    <div class="price-row">
                        <template v-if="parsed.finalPrice">
                            <span class="price-final">{{ fmt(parsed.finalPrice) }}</span>
                            <span class="price-unit">تومان</span>
                            <span v-if="parsed.basePrice && parsed.basePrice > parsed.finalPrice" class="price-old">{{
                                fmt(parsed.basePrice) }}</span>
                        </template>
                        <span v-else class="price-placeholder">قیمت نامشخص</span>
                    </div>

                    <div v-if="parsed.discountEnd" class="discount-date">
                        <i class="bi bi-clock-history"></i>
                        تا {{ parsed.discountEnd }}
                    </div>

                    <div class="stock-row" v-if="parsed.stock !== null">
                        <i class="bi" :class="parsed.stock > 0 ? 'bi-check-circle-fill' : 'bi-x-circle-fill'"
                            :style="{ color: parsed.stock > 0 ? '#10b981' : '#ef4444' }"></i>
                        <span>موجودی: <strong>{{ fmt(parsed.stock) }}</strong></span>
                    </div>

                    <div v-if="parsed.categories.length" class="cat-row">
                        <span v-for="c in parsed.categories" :key="c.id" class="cat-chip">{{ c.label }}</span>
                    </div>

                    <div v-if="parsed.description" class="desc-row">
                        {{ truncate(parsed.description, 160) }}
                    </div>

                    <!-- ویژگی‌های انتخاب‌شده -->
                    <div v-for="attr in attributesWithValues" :key="attr.id" class="variant-row">
                        <span class="vr-label">{{ attr.name }}:</span>
                        <span v-for="v in attributeValue[attr.id]" :key="v.id" class="color-chip">{{ v.value }}</span>
                    </div>

                    <details v-if="parsed.title" class="meta-preview">
                        <summary>اطلاعات سئو</summary>
                        <div class="meta-line"><b>Slug:</b> {{ slug }}</div>
                        <div class="meta-line"><b>Meta Title:</b> {{ metaTitle }}</div>
                        <div class="meta-line"><b>Meta Desc:</b> {{ metaDescription }}</div>
                    </details>
                </div>
            </div>

            <div class="guide">
                <h4><i class="bi bi-lightbulb"></i> راهنما</h4>
                <ol>
                    <li><b>خط ۱:</b> عنوان ← Enter</li>
                    <li><b>خط ۲:</b> قیمت — فرمت‌ها:</li>
                    <li><code>250000</code> ساده</li>
                    <li><code>250000-5%</code> درصد تخفیف</li>
                    <li><code>250000-200000</code> قیمت نهایی</li>
                    <li><code>250000-200000-1405/10/10</code> + تاریخ</li>
                    <li><code>250000-5%-1405/10/10</code> درصد + تاریخ</li>
                    <li><b>خط ۳:</b> موجودی ← Enter</li>
                    <li><b>خط ۴+:</b> دسته‌بندی‌ها ← Enter خالی</li>
                    <li><b>بعد:</b> توضیحات</li>
                </ol>
            </div>
        </aside>
    </div>
</template>

<script setup>
import { ref, reactive, computed, onMounted, nextTick } from 'vue'
import Treeselect from 'vue3-treeselect'
import 'vue3-treeselect/dist/vue3-treeselect.css'
import axios from 'axios'
import { toast } from 'vue3-toastify'
import { useAdmin } from '@/stores/modules/admin'

const store = useAdmin()
const checkPermission = store.checkPermission

/* ==================== STATE ==================== */
const currentInput = ref('')
const committedLines = ref([])
const inputRef = ref(null)
const fileInput = ref(null)

const images = ref([])
const loading = ref(false)
const errors = ref([])
const categoriesFlat = ref([])

/* ==================== ATTRIBUTES ==================== */
const attributes = ref([])
const selectedAttributes = ref([])
const attributeValue = reactive({})

/* ==================== PARSED ==================== */
const parsed = reactive({
    title: '',
    basePrice: null,
    finalPrice: null,
    discountPercent: null,
    discountEnd: '',
    stock: null,
    categories: [],
    description: '',
})

/* ==================== HELPERS ==================== */
const fmt = (n) => Number(n || 0).toLocaleString('fa-IR')
const truncate = (s, n) => (s?.length > n ? s.slice(0, n) + '…' : s)

const slug = computed(() =>
    (parsed.title || '')
        .trim()
        .toLowerCase()
        .replace(/[\s_]+/g, '-')
        .replace(/[^a-z0-9\u0600-\u06FF-]/g, '')
        .replace(/-+/g, '-')
        .replace(/^-+|-+$/g, '')
)

const metaTitle = computed(() =>
    parsed.title ? `${parsed.title} | خرید از داروخانه دکتر اسماعیلی` : ''
)
const metaDescription = computed(() =>
    parsed.title
        ? `خرید ${parsed.title} با بهترین قیمت و ارسال سریع از داروخانه دکتر اسماعیلی.`
        : ''
)

const previewImage = computed(() => images.value[0]?.url || '')

const discountLabel = computed(() => {
    if (parsed.basePrice && parsed.finalPrice && parsed.basePrice > parsed.finalPrice) {
        const pct =
            parsed.discountPercent ||
            Math.round((1 - parsed.finalPrice / parsed.basePrice) * 100)
        return `${pct}%`
    }
    return ''
})

const canSave = computed(() => parsed.title && parsed.basePrice && parsed.stock !== null)

/* ==================== NORMALIZERS ==================== */
const attributeNormalizer = (node) => ({ id: node.id, label: node.name })
const attributeValuesNormalizer = (node) => ({ id: node.id, label: node.value })

/* ==================== ویژگی‌های دارای مقدار ==================== */
const attributesWithValues = computed(() =>
    selectedAttributes.value
        .map((id) => attributes.value.find((a) => a.id == id))
        .filter((a) => a && attributeValue[a.id]?.length)
)

function attrName(id) {
    const found = attributes.value.find((attr) => attr.id == id)
    return found ? found.name : 'گزینه‌ها'
}

function attributeOptionsFor(attributeId) {
    const found = attributes.value.find((attr) => attr.id == attributeId)
    return found ? found.values : []
}

/* ==================== خط فعال ==================== */
const currentLineType = computed(() => detectType(committedLines.value.length))

function detectType(index) {
    if (index === 0) return 'title'
    if (index === 1) return 'price'
    if (index === 2) return 'stock'

    const last = committedLines.value[committedLines.value.length - 1]
    if (last && last.type === 'category' && last.empty) return 'description'
    if (last && last.type === 'description') return 'description'
    return 'category'
}

const currentPlaceholder = computed(() => {
    switch (currentLineType.value) {
        case 'title': return 'عنوان محصول را وارد کنید...'
        case 'price': return 'مثلاً 250000 | 250000-5% | 250000-200000-1405/10/10'
        case 'stock': return 'موجودی (عدد)...'
        case 'category': return 'نام دسته‌بندی... یا Tab برای انتخاب اولین پیشنهاد'
        case 'description': return 'توضیحات کامل محصول...'
        default: return ''
    }
})

function lineIcon(type) {
    return {
        title: 'bi bi-type-h1',
        price: 'bi bi-currency-dollar',
        stock: 'bi bi-box-seam',
        category: 'bi bi-tag',
        description: 'bi bi-file-text',
    }[type] || 'bi bi-dot'
}

/* ==================== ساجشن دسته ==================== */
const suggestions = computed(() => {
    const q = currentInput.value.trim().toLowerCase()
    if (!q) return categoriesFlat.value.slice(0, 6)
    return categoriesFlat.value
        .filter((c) => c.label.toLowerCase().includes(q))
        .slice(0, 6)
})

/* ==================== 🐛 فیکس Enter موبایل ==================== */
let lastCommitTime = 0

function safeCommit() {
    const now = Date.now()
    if (now - lastCommitTime < 120) return
    lastCommitTime = now
    commitLine()
}

function onBeforeInput(e) {
    if (e.inputType === 'insertLineBreak' || e.inputType === 'insertParagraph') {
        e.preventDefault()
        safeCommit()
    }
}

function onBlurInput() {
    // فوکوس رو نگه‌دار — کاری نکن
}

function focusInput() {
    const el = inputRef.value
    if (!el) return
    setTimeout(() => {
        el.focus({ preventScroll: true })
        try {
            el.setSelectionRange(el.value.length, el.value.length)
        } catch (_) { }
        el.scrollIntoView({ behavior: 'smooth', block: 'nearest' })
    }, 50)
}

/* ==================== Backspace هوشمند ==================== */
function handleBackspace(e) {
    if (currentInput.value.length === 0 && committedLines.value.length > 0) {
        e.preventDefault()
        const last = committedLines.value.pop()
        if (!last) return

        if (last.type === 'title') parsed.title = ''
        else if (last.type === 'price') {
            parsed.basePrice = null
            parsed.finalPrice = null
            parsed.discountPercent = null
            parsed.discountEnd = ''
        } else if (last.type === 'stock') parsed.stock = null
        else if (last.type === 'category') {
            parsed.categories = parsed.categories.filter((c) => c.label !== last.value)
        } else if (last.type === 'description') parsed.description = ''

        currentInput.value = last.value ?? ''
        nextTick(() => focusInput())
    }
}

/* ==================== COMMIT ==================== */
function commitLine() {
    const raw = currentInput.value.trim()
    const type = currentLineType.value

    if (!raw) {
        if (type === 'category' && committedLines.value.length >= 4) {
            committedLines.value.push({ type: 'category', value: '', empty: true })
            currentInput.value = ''
            nextTick(() => focusInput())
        }
        return
    }

    if (type === 'title') {
        parsed.title = raw
        committedLines.value.push({ type: 'title', value: raw })
    } else if (type === 'price') {
        const p = parsePrice(raw)
        parsed.basePrice = p.base
        parsed.finalPrice = p.final
        parsed.discountPercent = p.percent
        parsed.discountEnd = p.endDate
        committedLines.value.push({ type: 'price', value: raw, raw: p })
    } else if (type === 'stock') {
        const n = parseInt(raw.replace(/\D/g, ''), 10)
        parsed.stock = isNaN(n) ? 0 : n
        committedLines.value.push({ type: 'stock', value: parsed.stock })
    } else if (type === 'category') {
        const found =
            categoriesFlat.value.find((c) => c.label.toLowerCase() === raw.toLowerCase()) ||
            suggestions.value[0]
        if (found && !parsed.categories.find((c) => c.id === found.id)) {
            parsed.categories.push(found)
            committedLines.value.push({ type: 'category', value: found.label })
        }
    } else if (type === 'description') {
        parsed.description = raw
        committedLines.value.push({ type: 'description', value: raw })
    }

    currentInput.value = ''
    nextTick(() => focusInput())
}

function pickCategory(cat) {
    if (parsed.categories.find((c) => c.id === cat.id)) return
    parsed.categories.push(cat)
    committedLines.value.push({ type: 'category', value: cat.label })
    currentInput.value = ''
    nextTick(() => focusInput())
}

function removeCategory(id) {
    const cat = parsed.categories.find((c) => c.id === id)
    if (!cat) return
    parsed.categories = parsed.categories.filter((c) => c.id !== id)
    committedLines.value = committedLines.value.filter(
        (l) => !(l.type === 'category' && l.value === cat.label)
    )
}

function handleTab() {
    if (currentLineType.value === 'category' && suggestions.value.length) {
        pickCategory(suggestions.value[0])
    }
}

/* ==================== PRICE PARSER ==================== */
function parsePrice(raw) {
    const parts = raw.split('-').map((s) => s.trim())
    const base = parseInt(parts[0]?.replace(/\D/g, ''), 10) || 0

    let final = base
    let percent = null
    let endDate = ''

    const isPercent = (s) => /%/.test(s)
    const isDate = (s) => /\d{2,4}\/\d{1,2}\/\d{1,2}/.test(s)
    const toNum = (s) => parseInt(s.replace(/[^\d]/g, ''), 10)

    for (let i = 1; i < parts.length; i++) {
        const p = parts[i]
        if (!p) continue
        if (isDate(p)) endDate = p
        else if (isPercent(p)) {
            const n = toNum(p)
            if (!isNaN(n)) percent = n
        } else {
            const n = toNum(p)
            if (!isNaN(n) && n > 0) final = n
        }
    }

    if (percent !== null && final === base) {
        final = Math.round(base * (1 - percent / 100))
    }
    if (percent === null && final !== base && base > 0) {
        percent = Math.round((1 - final / base) * 100)
    }

    return { base, final, percent, endDate }
}

/* ==================== تصاویر ==================== */
function onFiles(e) {
    Array.from(e.target.files || []).forEach((f) => {
        images.value.push({ file: f, url: URL.createObjectURL(f) })
    })
    e.target.value = ''
}
function onDrop(e) {
    Array.from(e.dataTransfer.files || [])
        .filter((f) => f.type.startsWith('image/'))
        .forEach((f) => {
            images.value.push({ file: f, url: URL.createObjectURL(f) })
        })
}
function removeImage(i) {
    URL.revokeObjectURL(images.value[i].url)
    images.value.splice(i, 1)
}

/* ==================== LOAD ==================== */
onMounted(async () => {
    try {
        const [catRes, attrRes] = await Promise.all([
            axios.get('/categories'),
            axios.get('/attributes'),
        ])
        categoriesFlat.value = flattenCats(catRes.data.data || [])
        attributes.value = attrRes.data.data || []

        attributes.value.forEach((attr) => {
            if (!attributeValue[attr.id]) attributeValue[attr.id] = []
        })
    } catch (e) {
        console.warn('load failed', e)
    }
    nextTick(() => focusInput())
})

function flattenCats(list, parent = '') {
    let out = []
    for (const c of list) {
        const label = parent ? `${parent} › ${c.title}` : c.title
        out.push({ id: c.id, label })
        if (c.all_children?.length) out = out.concat(flattenCats(c.all_children, label))
    }
    return out
}

/* ==================== SAVE ==================== */
async function save() {
    errors.value = []
    if (!canSave.value) {
        errors.value.push('عنوان، قیمت و موجودی الزامی است.')
        return
    }

    loading.value = true
    try {
        const fd = new FormData()
        fd.append('title', parsed.title)
        fd.append('slug', slug.value)
        fd.append('price', parsed.basePrice)

        if (parsed.finalPrice && parsed.finalPrice < parsed.basePrice) {
            fd.append('discount_value', parsed.basePrice - parsed.finalPrice)
            fd.append('discount_type', 'fixed')
            if (parsed.discountEnd) fd.append('discount_end_at', parsed.discountEnd)
        }

        fd.append('stock', parsed.stock)
        fd.append('status', 'published')
        fd.append('sales_channel', 'both')
        fd.append('description', parsed.description)
        fd.append('meta_title', metaTitle.value)
        fd.append('meta_description', metaDescription.value)

        parsed.categories.forEach((c) => fd.append('categories[]', c.id))
        images.value.forEach((img, i) => {
            fd.append(i === 0 ? 'main_image' : 'images[]', img.file)
        })

        const res = await axios.post('/products', fd)
        const productId = res.data.id ?? res.data.data?.id

        if (attributesWithValues.value.length && productId) {
            await createVariants(productId)
        }

        toast.success('محصول با موفقیت ساخته شد ✅')
        resetAll()
    } catch (e) {
        if (e.response?.data?.errors) {
            Object.values(e.response.data.errors)
                .flat()
                .forEach((m) => errors.value.push(m))
        } else {
            errors.value.push('خطا در ارتباط با سرور')
        }
        toast.error('خطا در ذخیره محصول')
    } finally {
        loading.value = false
    }
}

/* ==================== ساخت تنوع‌ها ==================== */
async function createVariants(productId) {
    const activeLists = attributesWithValues.value.map((attr) =>
        attributeValue[attr.id].map((v) => ({ id: v.id, value: v.value }))
    )

    const combos = activeLists.reduce(
        (acc, list) => {
            const result = []
            for (const combo of acc) {
                for (const item of list) {
                    result.push([...combo, item])
                }
            }
            return result
        },
        [[]]
    )

    const vfd = new FormData()
    combos.forEach((values, index) => {
        vfd.append(`variants[${index}][price]`, parsed.basePrice)
        vfd.append(`variants[${index}][in_store_price]`, parsed.basePrice)
        vfd.append(`variants[${index}][stock]`, parsed.stock)
        vfd.append(`variants[${index}][discount_value]`, 0)
        vfd.append(`variants[${index}][discount_type]`, '')
        values.forEach((v) => {
            vfd.append(`variants[${index}][values][]`, v.id)
        })
    })

    await axios.post(`/product-variant/${productId}/variants`, vfd)
}

/* ==================== RESET ==================== */
function resetAll() {
    currentInput.value = ''
    committedLines.value = []
    Object.assign(parsed, {
        title: '',
        basePrice: null,
        finalPrice: null,
        discountPercent: null,
        discountEnd: '',
        stock: null,
        categories: [],
        description: '',
    })
    images.value.forEach((i) => URL.revokeObjectURL(i.url))
    images.value = []
    selectedAttributes.value = []
    Object.keys(attributeValue).forEach((k) => {
        attributeValue[k] = []
    })
    errors.value = []
    nextTick(() => focusInput())
}
</script>

<style scoped>
/* ==================== LAYOUT ==================== */
.qpc-wrap {
    display: grid;
    grid-template-columns: 9fr 3fr;
    gap: 20px;
    padding: 20px;
    max-width: 1400px;
    margin: 0 auto;
    direction: rtl;
    font-family: 'Vazirmatn', system-ui, sans-serif;
    min-height: 100vh;
    background: #f8fafc;
}

/* ==================== PREVIEW ==================== */
.qpc-preview {
    position: sticky;
    top: 20px;
    height: fit-content;
    align-self: start;
}

.preview-header {
    display: flex;
    align-items: center;
    gap: 8px;
    margin-bottom: 12px;
    font-weight: 600;
    color: #475569;
    font-size: 14px;
}

.badge-live {
    background: #10b981;
    color: #fff;
    font-size: 11px;
    padding: 2px 8px;
    border-radius: 10px;
    animation: pulse 1.5s infinite;
}

@keyframes pulse {

    0%,
    100% {
        opacity: 1;
    }

    50% {
        opacity: 0.5;
    }
}

.product-card {
    background: #fff;
    border-radius: 16px;
    overflow: hidden;
    box-shadow: 0 4px 20px rgba(0, 0, 0, 0.08);
}

.product-card.empty {
    opacity: 0.85;
}

.card-image {
    position: relative;
    aspect-ratio: 1;
    background: #f1f5f9;
    display: flex;
    align-items: center;
    justify-content: center;
}

.card-image img {
    width: 100%;
    height: 100%;
    object-fit: cover;
}

.img-placeholder {
    display: flex;
    flex-direction: column;
    align-items: center;
    gap: 8px;
    color: #94a3b8;
}

.img-placeholder i {
    font-size: 40px;
}

.discount-badge {
    position: absolute;
    top: 10px;
    left: 10px;
    background: #ef4444;
    color: #fff;
    font-weight: 700;
    font-size: 13px;
    padding: 4px 10px;
    border-radius: 8px;
}

.card-body {
    padding: 14px;
}

.product-title {
    font-size: 17px;
    font-weight: 700;
    color: #0f172a;
    margin: 0 0 10px;
    line-height: 1.5;
    min-height: 26px;
}

.price-row {
    display: flex;
    align-items: baseline;
    gap: 6px;
    flex-wrap: wrap;
    margin-bottom: 10px;
}

.price-final {
    font-size: 22px;
    font-weight: 800;
    color: #0f172a;
}

.price-unit {
    font-size: 13px;
    color: #64748b;
}

.price-old {
    font-size: 14px;
    color: #94a3b8;
    text-decoration: line-through;
}

.price-placeholder {
    color: #94a3b8;
    font-size: 14px;
}

.discount-date {
    background: #fef3c7;
    color: #92400e;
    font-size: 12px;
    padding: 4px 8px;
    border-radius: 6px;
    display: inline-flex;
    align-items: center;
    gap: 4px;
    margin-bottom: 8px;
}

.stock-row {
    display: flex;
    align-items: center;
    gap: 6px;
    font-size: 13px;
    color: #475569;
    margin-bottom: 10px;
}

.cat-row {
    display: flex;
    flex-wrap: wrap;
    gap: 6px;
    margin-bottom: 10px;
}

.cat-chip {
    background: #eff6ff;
    color: #1d4ed8;
    font-size: 11px;
    padding: 3px 10px;
    border-radius: 12px;
}

.desc-row {
    font-size: 12px;
    color: #64748b;
    line-height: 1.7;
    padding-top: 8px;
    border-top: 1px dashed #e2e8f0;
    margin-bottom: 8px;
}

.variant-row {
    display: flex;
    align-items: center;
    gap: 6px;
    flex-wrap: wrap;
    margin-bottom: 6px;
}

.vr-label {
    font-size: 12px;
    color: #64748b;
}

.color-chip {
    font-size: 11px;
    padding: 3px 8px;
    border-radius: 6px;
    background: #f1f5f9;
    color: #334155;
}

.meta-preview {
    margin-top: 10px;
    padding-top: 10px;
    border-top: 1px dashed #e2e8f0;
    font-size: 11px;
}

.meta-preview summary {
    cursor: pointer;
    color: #64748b;
    font-weight: 600;
}

.meta-line {
    margin-top: 6px;
    color: #475569;
    word-break: break-all;
    direction: ltr;
    text-align: left;
}

.guide {
    margin-top: 16px;
    background: #f0f9ff;
    border: 1px solid #bae6fd;
    border-radius: 12px;
    padding: 14px;
}

.guide h4 {
    font-size: 14px;
    color: #0369a1;
    margin: 0 0 10px;
    display: flex;
    align-items: center;
    gap: 6px;
}

.guide ol {
    margin: 0;
    padding-right: 18px;
    font-size: 12px;
    color: #334155;
    line-height: 1.9;
}

.guide code {
    background: #fff;
    padding: 1px 6px;
    border-radius: 4px;
    font-size: 11px;
    color: #0369a1;
    direction: ltr;
    display: inline-block;
}

/* ==================== EDITOR ==================== */
.qpc-editor {
    background: #fff;
    border-radius: 16px;
    padding: 20px;
    box-shadow: 0 4px 20px rgba(0, 0, 0, 0.06);
    align-self: start;
}

.editor-header {
    margin-bottom: 16px;
    padding-bottom: 12px;
    border-bottom: 2px solid #f1f5f9;
}

.editor-header h3 {
    font-size: 18px;
    margin: 0;
    color: #0f172a;
    display: flex;
    align-items: center;
    gap: 8px;
}

.hint {
    font-size: 12px;
    color: #64748b;
    display: block;
    margin-top: 4px;
}

.smart-input {
    background: #f8fafc;
    border: 2px dashed #e2e8f0;
    border-radius: 12px;
    padding: 16px;
    min-height: 200px;
    margin-bottom: 20px;
    transition: border-color 0.2s;
}

.smart-input:focus-within {
    border-color: #3b82f6;
    background: #fff;
}

.line-committed,
.line-current {
    display: flex;
    align-items: center;
    gap: 10px;
    padding: 8px 0;
    font-size: 14px;
}

.line-committed {
    animation: fadeIn 0.2s ease;
}

@keyframes fadeIn {
    from {
        opacity: 0;
        transform: translateY(-4px);
    }

    to {
        opacity: 1;
        transform: translateY(0);
    }
}

.line-icon {
    width: 26px;
    height: 26px;
    border-radius: 6px;
    display: flex;
    align-items: center;
    justify-content: center;
    flex-shrink: 0;
    font-size: 13px;
}

.line-title .line-icon {
    background: #dbeafe;
    color: #1d4ed8;
}

.line-price .line-icon {
    background: #dcfce7;
    color: #15803d;
}

.line-stock .line-icon {
    background: #fef3c7;
    color: #b45309;
}

.line-category .line-icon {
    background: #ede9fe;
    color: #6d28d9;
}

.line-description .line-icon {
    background: #fce7f3;
    color: #be185d;
}

.line-content {
    flex: 1;
    color: #1e293b;
}

.line-content b {
    font-weight: 700;
}

.mini-badge {
    display: inline-block;
    background: #10b981;
    color: #fff;
    font-size: 10px;
    padding: 1px 6px;
    border-radius: 8px;
    margin-right: 4px;
}

.mini-badge.date {
    background: #f59e0b;
}

.line-current {
    padding: 8px 0;
    flex-wrap: wrap;
}

.smart-line-input {
    flex: 1;
    border: none;
    background: transparent;
    font-size: 16px;
    padding: 6px 0;
    outline: none;
    color: #0f172a;
    font-family: inherit;
    min-width: 120px;
}

.smart-line-input::placeholder {
    color: #94a3b8;
}

.cat-box {
    display: inline-block;
    background: #ede9fe;
    color: #6d28d9;
    padding: 3px 12px;
    border-radius: 6px;
    font-weight: 500;
    border: 2px solid #a78bfa;
}

.suggestions {
    flex-basis: 100%;
    display: flex;
    flex-wrap: wrap;
    gap: 6px;
    padding-top: 8px;
    margin-top: 4px;
    border-top: 1px dashed #cbd5e1;
}

.suggestion-chip {
    background: #fff;
    border: 1px solid #c7d2fe;
    color: #4338ca;
    font-size: 12px;
    padding: 4px 10px;
    border-radius: 16px;
    cursor: pointer;
    transition: all 0.15s;
    display: flex;
    align-items: center;
    gap: 4px;
    font-family: inherit;
}

.suggestion-chip:hover {
    background: #4338ca;
    color: #fff;
    border-color: #4338ca;
}

.selected-cats {
    margin-top: 12px;
    padding-top: 12px;
    border-top: 1px dashed #cbd5e1;
    display: flex;
    flex-wrap: wrap;
    gap: 6px;
    align-items: center;
}

.sc-label {
    font-size: 12px;
    color: #64748b;
}

.sc-chip {
    background: #ede9fe;
    color: #6d28d9;
    font-size: 12px;
    padding: 4px 10px;
    border-radius: 12px;
    display: inline-flex;
    align-items: center;
    gap: 6px;
}

.sc-chip i {
    cursor: pointer;
    opacity: 0.7;
}

.sc-chip i:hover {
    opacity: 1;
}

/* ==================== UPLOAD ==================== */
.upload-section,
.variant-section {
    margin-bottom: 20px;
    padding-top: 16px;
    border-top: 2px solid #f1f5f9;
}

.upload-section h4,
.variant-section h4 {
    font-size: 15px;
    margin: 0 0 12px;
    color: #0f172a;
    display: flex;
    align-items: center;
    gap: 6px;
}

.upload-box {
    border: 2px dashed #cbd5e1;
    border-radius: 10px;
    padding: 24px;
    text-align: center;
    cursor: pointer;
    transition: all 0.2s;
    background: #f8fafc;
}

.upload-box:hover {
    border-color: #3b82f6;
    background: #eff6ff;
}

.upload-box i {
    font-size: 32px;
    color: #64748b;
    display: block;
    margin-bottom: 6px;
}

.upload-box p {
    margin: 0;
    font-size: 13px;
    color: #64748b;
}

.thumbs {
    display: flex;
    flex-wrap: wrap;
    gap: 8px;
    margin-top: 12px;
}

.thumb {
    position: relative;
    width: 80px;
    height: 80px;
    border-radius: 8px;
    overflow: hidden;
    border: 2px solid #e2e8f0;
}

.thumb.main {
    border-color: #10b981;
}

.thumb img {
    width: 100%;
    height: 100%;
    object-fit: cover;
}

.main-tag {
    position: absolute;
    bottom: 0;
    left: 0;
    right: 0;
    background: #10b981;
    color: #fff;
    font-size: 10px;
    text-align: center;
    padding: 2px;
}

.remove-thumb {
    position: absolute;
    top: 2px;
    right: 2px;
    background: rgba(239, 68, 68, 0.9);
    color: #fff;
    width: 20px;
    height: 20px;
    border-radius: 50%;
    display: flex;
    align-items: center;
    justify-content: center;
    font-size: 11px;
    cursor: pointer;
}

/* ==================== VARIANTS ==================== */
.variant-section {
    display: flex;
    flex-direction: column;
    gap: 12px;
}

.vi-group {
    display: flex;
    flex-direction: column;
    gap: 6px;
}

.vi-group.full {
    width: 100%;
}

.vi-group label {
    font-size: 12px;
    color: #64748b;
    font-weight: 500;
}

.variant-note {
    background: #fef3c7;
    color: #92400e;
    font-size: 12px;
    padding: 8px 12px;
    border-radius: 8px;
    margin-top: 4px;
    display: flex;
    align-items: center;
    gap: 6px;
}

/* ==================== SAVE ==================== */
.save-bar {
    display: flex;
    gap: 10px;
    padding-top: 16px;
    border-top: 2px solid #f1f5f9;
}

.btn-save {
    flex: 1;
    background: linear-gradient(135deg, #3b82f6, #2563eb);
    color: #fff;
    border: none;
    padding: 12px;
    border-radius: 10px;
    font-size: 15px;
    font-weight: 600;
    cursor: pointer;
    display: flex;
    align-items: center;
    justify-content: center;
    gap: 8px;
    font-family: inherit;
}

.btn-save:hover:not(:disabled) {
    transform: translateY(-1px);
}

.btn-save:disabled {
    opacity: 0.5;
    cursor: not-allowed;
}

.btn-reset {
    background: #f1f5f9;
    color: #64748b;
    border: none;
    padding: 12px 16px;
    border-radius: 10px;
    cursor: pointer;
    font-family: inherit;
    font-size: 14px;
    display: flex;
    align-items: center;
    gap: 4px;
}

.btn-reset:hover {
    background: #e2e8f0;
}

.spinner {
    width: 16px;
    height: 16px;
    border: 2px solid rgba(255, 255, 255, 0.4);
    border-top-color: #fff;
    border-radius: 50%;
    animation: spin 0.7s linear infinite;
}

@keyframes spin {
    to {
        transform: rotate(360deg);
    }
}

.errors-box {
    margin-top: 16px;
    background: #fef2f2;
    border: 1px solid #fecaca;
    border-radius: 10px;
    padding: 12px;
}

.errors-box h5 {
    margin: 0 0 8px;
    color: #b91c1c;
    font-size: 13px;
    display: flex;
    align-items: center;
    gap: 4px;
}

.errors-box ul {
    margin: 0;
    padding-right: 18px;
    font-size: 12px;
    color: #7f1d1d;
}

/* ==================== RESPONSIVE ==================== */
@media (max-width: 992px) {
    .qpc-wrap {
        grid-template-columns: 1fr;
        padding: 12px;
    }

    .qpc-preview {
        position: static;
        order: 2;
    }

    .qpc-editor {
        order: 1;
    }

    .product-card {
        max-width: 420px;
        margin: 0 auto;
    }
}

@media (max-width: 576px) {
    .qpc-wrap {
        padding: 8px;
        gap: 12px;
    }

    .qpc-editor {
        padding: 14px;
        border-radius: 12px;
    }

    .editor-header h3 {
        font-size: 16px;
    }

    .smart-input {
        padding: 10px;
    }

    .line-committed,
    .line-current {
        font-size: 14px;
    }

    .price-final {
        font-size: 19px;
    }

    .save-bar {
        flex-direction: column;
    }

    .guide ol {
        font-size: 11px;
    }

    /* 🐛 فیکس موبایل */
    .smart-line-input {
        font-size: 16px !important;
        min-width: 0;
        width: 100%;
    }

    .line-current {
        flex-wrap: wrap;
    }

    .suggestions {
        flex-basis: 100%;
        order: 3;
    }

    .suggestion-chip {
        padding: 8px 14px;
        font-size: 13px;
    }

    /* Treeselect روی موبایل ارتفاع مناسب */
    .vi-group :deep(.vue-treeselect__control) {
        min-height: 44px;
    }
}
</style>