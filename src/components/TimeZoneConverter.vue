<script setup>
import { ref, computed, watch } from 'vue';
import { DateTime } from 'luxon';
import html2canvas from 'html2canvas';

// --- State ---
const baseZone = 'Africa/Lagos';

// UI State
const selectedDate = ref(DateTime.now().setZone(baseZone));
const showDateModal = ref(false);
const customDateInput = ref('');
const dateStripStart = ref(DateTime.now().setZone(baseZone).startOf('day'));

// Clock State
// Clock State
const inputHour = ref('12');
const inputMinute = ref('00');
const period = ref('PM');

// Ensure minute is snapped to 00, 15, 30, 45 initially
// (Removed dynamic snapping logic since we are defaulting to 00)

// Zone Definitions
const locations = [
    { label: 'Nigeria (Base)', zone: 'Africa/Lagos', flag: '🇳🇬', code: 'ng' },
    { label: 'India', zone: 'Asia/Kolkata', flag: '🇮🇳', code: 'in' },
    { label: 'Egypt', zone: 'Africa/Cairo', flag: '🇪🇬', code: 'eg' },
    { label: 'Iraq', zone: 'Asia/Baghdad', flag: '🇮🇶', code: 'iq' },
    { label: 'Germany', zone: 'Europe/Berlin', flag: '🇩🇪', code: 'de' },
    { label: 'Bangladesh', zone: 'Asia/Dhaka', flag: '🇧🇩', code: 'bd' },
    { label: 'Indonesia', zone: 'Asia/Jakarta', flag: '🇮🇩', code: 'id' },
    { label: 'System Time', zone: 'local', flag: '💻', code: 'un' },
];
// --- Computed: Date Strip ---
const dateStrip = computed(() => {
    const days = [];
    const start = dateStripStart.value;
    for (let i = 0; i < 14; i++) {
        const dt = start.plus({ days: i });
        days.push({
            dateObj: dt,
            dayName: dt.toFormat('ccc'), // Mon
            dayNum: dt.toFormat('dd'),   // 11
            fullDate: dt.toFormat('yyyy-MM-dd'),
            isSelected: dt.hasSame(selectedDate.value, 'day')
        });
    }
    return days;
});

// --- Computed: Clock Logic ---
const clockHours = [
    { val: 12, label: '12', type: 'major' },
    { val: 1, label: '', type: 'minor' },
    { val: 2, label: '', type: 'minor' },
    { val: 3, label: '3', type: 'major' },
    { val: 4, label: '', type: 'minor' },
    { val: 5, label: '', type: 'minor' },
    { val: 6, label: '6', type: 'major' },
    { val: 7, label: '', type: 'minor' },
    { val: 8, label: '', type: 'minor' },
    { val: 9, label: '9', type: 'major' },
    { val: 10, label: '', type: 'minor' },
    { val: 11, label: '', type: 'minor' },
];

const minuteOptions = ["00", "15", "30", "45"];

// Helper to position numbers on circle
const getClockPosition = (index) => {
    const radius = 98;
    const angleRad = (index * 30 - 90) * (Math.PI / 180);

    // Calculate Standard Position
    let x = Math.round(radius * Math.cos(angleRad));
    let y = Math.round(radius * Math.sin(angleRad));

    // Optional: Add manual offsets here to tweak specific numbers
    // 0 = 12, 3 = 3, 6 = 6, 9 = 9
    const manualAdjustments = {
        0: { x: -22, y: -19 },  // Tweak 12 o'clock
        // 3: { x: 5, y: 0 }, example
    };

    if (manualAdjustments[index]) {
        x += manualAdjustments[index].x;
        y += manualAdjustments[index].y;
    } else {
        // Apply the same offset to everything else to align with 12
        x += -22;
        y += -19;
    }

    return {
        transform: `translate(${x}px, ${y}px)`
    };
};

// --- Computed: Results ---
const scheduledTimes = computed(() => {
    // Convert 12h + AM/PM to 24h
    let h = parseInt(inputHour.value);
    if (period.value === 'PM' && h !== 12) h += 12;
    if (period.value === 'AM' && h === 12) h = 0;

    const m = parseInt(inputMinute.value);
    const baseDT = selectedDate.value.set({ hour: h, minute: m });

    return locations.map(loc => {
        const zonedDT = baseDT.setZone(loc.zone);

        // Formatting
        const timeStr = zonedDT.toFormat('h:mm a');
        const dayStr = zonedDT.toFormat('cccc');

        // Check day diff relative to BASE
        const baseDayStart = baseDT.startOf('day');
        const targetDayStart = zonedDT.startOf('day');
        const dayDiff = targetDayStart.diff(baseDayStart, 'days').days;

        let dayDiffLabel = '';
        if (dayDiff > 0.5) dayDiffLabel = 'Tomorrow';
        else if (dayDiff < -0.5) dayDiffLabel = 'Yesterday';

        // Offset
        const diffInMinutes = zonedDT.offset - baseDT.offset;
        const diffInHours = diffInMinutes / 60;
        const diffSign = diffInHours >= 0 ? '+' : '';
        const offsetLabel = diffInHours === 0 ? 'Same Time' : `${diffSign}${diffInHours} HRS`;

        return {
            ...loc,
            time: timeStr,
            day: dayStr,
            isBase: loc.zone === baseZone,
            dayDiffLabel,
            offsetLabel,
            region: loc.zone.includes('/') ? loc.zone.split('/')[1].replace('_', ' ') : 'Local System'
        };
    });
});

// --- Methods ---
const selectDay = (dayItem) => selectedDate.value = dayItem.dateObj;
const selectHour = (h) => inputHour.value = h.toString();
const selectMinute = (m) => inputMinute.value = m;

const openDateModal = () => {
    customDateInput.value = selectedDate.value.toFormat('yyyy-MM-dd');
    showDateModal.value = true;
};

const applyCustomDate = () => {
    if (!customDateInput.value) return;
    const newDate = DateTime.fromFormat(customDateInput.value, 'yyyy-MM-dd').setZone(baseZone);
    if (newDate.isValid) {
        selectedDate.value = newDate;
        dateStripStart.value = newDate.minus({ days: 1 }); // Start strip slightly before selected date context
        showDateModal.value = false;
    }
};

const copied = ref(false);
const copySchedule = () => {
    const text = scheduledTimes.value
        .map(t => `${t.flag} ${t.label}: ${t.time}`)
        .join('\n');
    navigator.clipboard.writeText(text).then(() => {
        copied.value = true;
        setTimeout(() => copied.value = false, 2000);
    });
};

const captureRef = ref(null);
const captureSchedule = async () => {
    if (!captureRef.value) return;
    try {
        const canvas = await html2canvas(captureRef.value, {
            backgroundColor: '#FFFFFF',
            scale: 4,
            useCORS: true,
            ignoreElements: (element) => element.hasAttribute('data-html2canvas-ignore'),
            onclone: (clonedDoc) => {
                const clonedEl = clonedDoc.querySelector('[data-capture-target]');
                if (clonedEl) {
                    clonedEl.style.maxWidth = '576px'; // max-w-xl equivalent
                    clonedEl.style.margin = '0 auto';
                }
            }
        });
        const link = document.createElement('a');
        link.download = `schedule-${DateTime.now().toFormat('yyyyMMdd-HHmm')}.png`;
        link.href = canvas.toDataURL('image/png');
        link.click();
    } catch (err) {
        console.error('Capture failed:', err);
        alert('Failed to capture.');
    }
};
</script>

<template>
    <div class="min-h-screen bg-white text-slate-800 p-4 md:p-8 font-sans w-full">
        <div class="w-full space-y-8 max-w-7xl mx-auto">

            <!-- Header -->
            <header class="flex justify-between items-center px-4">
                <h1 class="text-3xl font-bold text-slate-900 flex items-center gap-2">
                    <span
                        class="w-10 h-10 bg-[#5B4DFF] rounded-xl text-white flex items-center justify-center text-xl font-black">Z</span>
                    Time Converter
                </h1>

            </header>

            <!-- Layout: Two Columns (Controls | Results) -->
            <div class="flex flex-col  gap-8">

                <!-- Controls Column -->
                <div class="flex-1 space-y-8">

                    <!-- Date Strip -->
                    <div class="bg-slate-50 p-6 rounded-[2rem]">
                        <div class="flex items-center justify-between mb-4">
                            <h2 class="text-xl font-bold text-slate-900 flex items-center gap-2">
                                <svg width="20" height="20" viewBox="0 0 24 24" fill="none" stroke="currentColor"
                                    stroke-width="2">
                                    <rect x="3" y="4" width="18" height="18" rx="2" ry="2"></rect>
                                    <line x1="16" y1="2" x2="16" y2="6"></line>
                                    <line x1="8" y1="2" x2="8" y2="6"></line>
                                    <line x1="3" y1="10" x2="21" y2="10"></line>
                                </svg>
                                Select Date
                            </h2>
                            <button @click="openDateModal"
                                class="text-slate-400 hover:text-[#5B4DFF] hover:bg-[#5B4DFF]/5 p-2 rounded-full transition-all"
                                title="Select Specific Date">
                                <svg xmlns="http://www.w3.org/2000/svg" width="20" height="20" viewBox="0 0 24 24"
                                    fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round"
                                    stroke-linejoin="round">
                                    <circle cx="12" cy="12" r="10" />
                                    <line x1="12" y1="8" x2="12" y2="16" />
                                    <line x1="8" y1="12" x2="16" y2="12" />
                                </svg>
                            </button>
                        </div>
                        <div class="flex gap-3 overflow-x-auto pb-2 no-scrollbar">
                            <button v-for="day in dateStrip" :key="day.fullDate" @click="selectDay(day)"
                                class="cursor-pointer relative flex-shrink-0 w-20 h-24 rounded-2xl flex flex-col items-center justify-center transition-all group"
                                :class="day.isSelected ? 'bg-[#5B4DFF] text-white shadow-lg shadow-[#5B4DFF]/30' : 'bg-white text-slate-400 hover:bg-slate-200'">
                                <span class="text-2xl font-bold mb-1">{{ day.dayNum }}</span>
                                <span class="text-[10px] font-bold uppercase tracking-wider">{{ day.dayName }}</span>
                                <!-- Notch -->
                                <div class="absolute -left-1.5 top-1/2 -translate-y-1/2 w-3 h-3 rounded-full"
                                    :class="day.isSelected ? 'bg-slate-50' : 'bg-slate-50'"></div>
                                <div class="absolute -right-1.5 top-1/2 -translate-y-1/2 w-3 h-3 rounded-full"
                                    :class="day.isSelected ? 'bg-slate-50' : 'bg-slate-50'"></div>
                            </button>
                        </div>
                    </div>

                    <!-- Clock & Time Selection -->
                    <div class="bg-slate-50 p-6 rounded-[2rem] flex flex-col items-center">
                        <div class="flex items-center justify-between w-full mb-6">
                            <h2 class="text-xl font-bold text-slate-900 flex items-center gap-2">
                                <svg width="20" height="20" viewBox="0 0 24 24" fill="none" stroke="currentColor"
                                    stroke-width="2">
                                    <circle cx="12" cy="12" r="10"></circle>
                                    <polyline points="12 6 12 12 16 14"></polyline>
                                </svg>
                                Select Time
                            </h2>
                            <div
                                class="bg-[#5B4DFF]/10 text-[#5B4DFF] px-4 py-1.5 rounded-full text-base font-black tracking-wide border border-[#5B4DFF]/20 shadow-sm">
                                {{ inputHour }}:{{ inputMinute }} {{ period }}
                            </div>
                        </div>

                        <!-- Period Toggle -->
                        <div class="flex bg-white p-1 rounded-xl shadow-sm mb-8 w-64 border border-slate-100">
                            <button @click="period = 'AM'"
                                class="cursor-pointer flex-1 py-2 rounded-lg text-sm font-bold transition-all"
                                :class="period === 'AM' ? 'bg-[#5B4DFF] text-white shadow-md' : 'text-slate-400'">AM</button>
                            <button @click="period = 'PM'"
                                class="cursor-pointer flex-1 py-2 rounded-lg text-sm font-bold transition-all"
                                :class="period === 'PM' ? 'bg-[#5B4DFF] text-white shadow-md' : 'text-slate-400'">PM</button>
                        </div>

                        <!-- Analog Clock Face -->
                        <div
                            class="relative w-64 h-64 rounded-full bg-white shadow-xl shadow-slate-200/50 border-4 border-slate-100 mb-8 select-none">
                            <!-- Center Dot -->
                            <div class="absolute inset-0 m-auto w-3 h-3 bg-[#5B4DFF] rounded-full z-10"></div>

                            <!-- Hours -->
                            <div class="absolute w-full h-full">
                                <button v-for="(h, index) in clockHours" :key="h.val" @click="selectHour(h.val)"
                                    class="cursor-pointer absolute left-1/2 top-1/2 w-10 h-10 -ml-5 -mt-5 flex items-center justify-center rounded-full transition-all"
                                    :class="[
                                        inputHour == h.val
                                            ? 'bg-[#5B4DFF] text-white z-20 shadow-lg'
                                            : 'text-slate-400 hover:bg-slate-100'
                                    ]" :style="getClockPosition(index)">
                                    <!-- Display Number if Major, Dot if Minor -->
                                    <span v-if="h.type === 'major'" class="cursor-pointer text-lg font-bold">{{ h.label
                                    }}</span>
                                    <div v-else
                                        class="cursor-pointer w-1.5 h-1.5 rounded-full bg-slate-300 transition-colors"
                                        :class="inputHour == h.val ? 'bg-white' : ''"></div>
                                </button>
                            </div>
                        </div>

                        <!-- Minute Options -->
                        <div class="w-full">
                            <p class="text-center text-xs font-bold text-slate-400 uppercase tracking-widest mb-3">
                                Minutes</p>
                            <div class="grid grid-cols-4 gap-3 max-w-md mx-auto">
                                <button v-for="m in minuteOptions" :key="m" @click="selectMinute(m)"
                                    class="py-2 rounded-xl font-mono text-sm font-bold border-2 transition-all"
                                    :class="inputMinute === m ? 'border-[#5B4DFF] text-[#5B4DFF] bg-[#5B4DFF]/5' : 'border-slate-200 text-slate-400 bg-white hover:border-slate-300'">
                                    :{{ m }}
                                </button>
                            </div>
                        </div>

                    </div>
                </div>

                <!-- Result Column (Captured) -->
                <div class="flex-1">
                    <div ref="captureRef" data-capture-target="true"
                        class="rounded-[2.5rem] p-8 border shadow-2xl h-full"
                        style="background-color: #ffffff; border-color: #f1f5f9; box-shadow: 0 25px 50px -12px rgba(226, 232, 240, 0.5); color: #1e293b;">
                        <div class="flex items-end justify-between mb-8 pb-6 border-b" style="border-color: #f1f5f9;">
                            <div>
                                <h2 class="text-2xl font-bold tracking-tight flex items-center gap-2"
                                    style="color: #0f172a;">
                                    <img src="https://flagcdn.com/w80/ng.png" alt="Nigeria"
                                        class="w-8 h-8 rounded-full object-cover border" style="border-color: #f1f5f9;">
                                    Nigeria
                                </h2>
                                <p class="font-medium text-sm mt-1 pl-10" style="color: #64748b;">
                                    {{ selectedDate.toFormat('cccc, LLL dd') }}
                                </p>
                            </div>
                            <div class="text-right flex flex-col items-end">
                                <!-- Action Buttons -->
                                <div class="flex gap-2 mb-2" data-html2canvas-ignore>
                                    <button @click.stop="copySchedule"
                                        class="text-xs font-bold flex items-center gap-1 px-3 py-1.5 rounded-lg transition-colors border"
                                        :class="copied ? 'bg-green-50 text-green-600 border-green-200' : 'bg-[#5B4DFF]/5 text-[#5B4DFF] border-[#5B4DFF]/10 hover:bg-[#5B4DFF]/10'">
                                        <svg v-if="!copied" xmlns="http://www.w3.org/2000/svg" width="14" height="14"
                                            viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"
                                            stroke-linecap="round" stroke-linejoin="round">
                                            <rect x="9" y="9" width="13" height="13" rx="2" ry="2"></rect>
                                            <path d="M5 15H4a2 2 0 0 1-2-2V4a2 2 0 0 1 2-2h9a2 2 0 0 1 2 2v1"></path>
                                        </svg>
                                        <svg v-else xmlns="http://www.w3.org/2000/svg" width="14" height="14"
                                            viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"
                                            stroke-linecap="round" stroke-linejoin="round">
                                            <polyline points="20 6 9 17 4 12"></polyline>
                                        </svg>
                                        {{ copied ? 'Copied!' : 'Copy' }}
                                    </button>

                                    <button @click.stop="captureSchedule"
                                        class="text-xs font-bold flex items-center gap-1 px-3 py-1.5 rounded-lg transition-colors border"
                                        style="color: #5B4DFF; border-color: rgba(91, 77, 255, 0.1); background-color: rgba(91, 77, 255, 0.05);">
                                        <svg xmlns="http://www.w3.org/2000/svg" width="14" height="14"
                                            viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
                                            <path d="M21 15v4a2 2 0 0 1-2 2H5a2 2 0 0 1-2-2v-4" />
                                            <polyline points="7 10 12 15 17 10" />
                                            <line x1="12" y1="15" x2="12" y2="3" />
                                        </svg>
                                        Export
                                    </button>
                                </div>

                                <span class="block text-4xl font-black" style="color: #5B4DFF;">{{
                                    scheduledTimes[0]?.time }}</span>
                                <span class="text-xs font-bold uppercase tracking-widest" style="color: #94a3b8;">Base
                                    Time (Lagos)</span>
                            </div>
                        </div>

                        <!-- Results Grid (User List Style) -->
                        <div class="grid grid-cols-1 md:grid-cols-2 gap-4">
                            <div v-for="item in scheduledTimes.slice(1)" :key="item.zone"
                                class=" flex items-center p-4 rounded-3xl border transition-all duration-300 group hover:shadow-lg"
                                style="background-color: #ffffff; border-color: #f1f5f9;">
                                <!-- Avatar -->
                                <div class="w-12 h-12 rounded-full flex items-center justify-center flex-shrink-0 border overflow-hidden"
                                    style="background-color: #f8fafc; border-color: #f1f5f9;">
                                    <img :src="`https://flagcdn.com/w80/${item.code}.png`" :alt="item.label"
                                        class="w-full h-full object-cover opacity-90" />
                                </div>

                                <!-- Middle -->
                                <div class="ml-4 flex-1 min-w-0">
                                    <h3 class="text-sm font-bold truncate" style="color: #0f172a;">{{ item.label }}</h3>
                                    <p class="text-xs font-medium truncate pb-1" style="color: #94a3b8;">{{ item.region
                                        }}
                                    </p>
                                </div>

                                <!-- Right -->
                                <div class="text-right pl-3">
                                    <div class="text-sm font-black whitespace-nowrap transition-colors"
                                        style="color: #0f172a;">
                                        {{ item.time }}
                                    </div>
                                    <div class="text-[10px] font-bold mt-0.5 whitespace-nowrap flex flex-col items-end"
                                        style="color: #94a3b8;">
                                        <span>{{ item.offsetLabel }}</span>
                                        <span v-if="item.dayDiffLabel" style="color: #f97316;">{{ item.dayDiffLabel
                                            }}</span>
                                    </div>
                                </div>
                            </div>
                        </div>

                        <!-- Footer -->
                        <div class="mt-8 text-center opacity-30">
                            <span class="text-xs font-black tracking-[0.3em]" style="color: #cbd5e1;">ZONESYNC</span>
                        </div>
                    </div>
                </div>

                <!-- Date Selection Modal -->
                <div v-if="showDateModal"
                    class="fixed inset-0 z-50 flex items-center justify-center p-4 bg-slate-900/20 backdrop-blur-sm"
                    @click.self="showDateModal = false">
                    <div
                        class="bg-white rounded-[2rem] p-8 shadow-2xl max-w-sm w-full animate-in fade-in zoom-in duration-200">
                        <h3 class="text-xl font-bold text-slate-900 mb-6">Select Specific Date</h3>
                        <input type="date" v-model="customDateInput"
                            class="w-full text-lg font-bold p-4 rounded-xl border-2 border-slate-100 focus:border-[#5B4DFF] outline-none text-slate-700 mb-6 bg-slate-50" />
                        <div class="flex gap-3">
                            <button @click="showDateModal = false"
                                class="flex-1 py-3 font-bold text-slate-500 hover:bg-slate-50 rounded-xl transition-colors">Cancel</button>
                            <button @click="applyCustomDate"
                                class="flex-1 py-3 font-bold text-white bg-[#5B4DFF] hover:bg-[#4839eb] rounded-xl shadow-lg shadow-[#5B4DFF]/30 transition-all">Select</button>
                        </div>
                    </div>
                </div>

            </div>
        </div>
    </div>
</template>

<style scoped>
.no-scrollbar::-webkit-scrollbar {
    display: none;
}

.no-scrollbar {
    -ms-overflow-style: none;
    scrollbar-width: none;
}
</style>
