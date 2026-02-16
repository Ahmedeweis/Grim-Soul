<script setup>
import { ref, computed, watch } from 'vue';
import { DateTime } from 'luxon';
import html2canvas from 'html2canvas';

// --- State ---
const baseDefaultZone = 'Africa/Lagos'; // Fallback / Initial ref for calculations if needed, but we use computed baseZone now

// UI State
const selectedDate = ref(DateTime.now().setZone(baseDefaultZone));
const showDateModal = ref(false);
const customDateInput = ref('');
const dateStripStart = ref(DateTime.now().setZone(baseDefaultZone).startOf('day'));

// Clock State
// Clock State
const inputHour = ref('12');
const inputMinute = ref('00');
const period = ref('PM');

// Ensure minute is snapped to 00, 15, 30, 45 initially
// (Removed dynamic snapping logic since we are defaulting to 00)

// Zone Definitions
const locations = ref([
    { label: 'Nigeria (Base)', zone: 'Africa/Lagos', flag: '🇳🇬', code: 'ng' },
    { label: 'India', zone: 'Asia/Kolkata', flag: '🇮🇳', code: 'in' },
    { label: 'Egypt', zone: 'Africa/Cairo', flag: '🇪🇬', code: 'eg' },
    { label: 'Iraq', zone: 'Asia/Baghdad', flag: '🇮🇶', code: 'iq' },
    { label: 'Germany', zone: 'Europe/Berlin', flag: '🇩🇪', code: 'de' },
    { label: 'Bangladesh', zone: 'Asia/Dhaka', flag: '🇧🇩', code: 'bd' },
    { label: 'Indonesia', zone: 'Asia/Jakarta', flag: '🇮🇩', code: 'id' },
    { label: 'System Time', zone: 'local', flag: '💻', code: 'un' },
]);

// --- State ---
const baseCountry = ref('Nigeria'); // Default to Nigeria
const allCountries = ref([]);
const showCountryModal = ref(false);
const searchQuery = ref('');
const isLoadingCountries = ref(false);


// Toast State
const toast = ref({
    show: false,
    message: '',
    type: 'success' // success, error, warning
});

const triggerToast = (msg, type = 'success') => {
    toast.value.message = msg;
    toast.value.type = type;
    toast.value.show = true;
    setTimeout(() => {
        toast.value.show = false;
    }, 3000);
};
const baseZone = computed(() => {
    return baseCountry.value === 'Egypt' ? 'Africa/Cairo' : 'Africa/Lagos';
});

// --- Computed: Filtered Countries ---
const filteredCountries = computed(() => {
    if (!searchQuery.value) return allCountries.value;
    const query = searchQuery.value.toLowerCase().trim();
    return allCountries.value.filter(c => {
        const nameMatch = c.name.common.toLowerCase().includes(query);
        const capitalMatch = c.capital && c.capital.some(cap => cap.toLowerCase().includes(query));
        // Optional: Check alternative spellings if available for better results
        const altMatch = c.altSpellings && c.altSpellings.some(alt => alt.toLowerCase().includes(query));

        return nameMatch || capitalMatch || altMatch;
    });
});
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

    return locations.value.map(loc => {
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
            isBase: loc.zone === baseZone.value,
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
    const newDate = DateTime.fromFormat(customDateInput.value, 'yyyy-MM-dd').setZone(baseZone.value);
    if (newDate.isValid) {
        selectedDate.value = newDate;
        dateStripStart.value = newDate.minus({ days: 1 }); // Start strip slightly before selected date context
        showDateModal.value = false;
    }
};

const copied = ref(false);
const copySchedule = () => {
    const text = scheduledTimes.value
        .map(t => {
            const flagDisplay = t.flag && t.flag.startsWith('http') ? '' : t.flag;
            return `${flagDisplay} ${t.label}: ${t.time}`.trim();
        })
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
        triggerToast(`Export failed: ${err.message}`, 'error');
    }
}


// --- Methods: Country Management ---
const fetchCountries = async () => {
    if (allCountries.value.length > 0) return; // Already fetched
    isLoadingCountries.value = true;
    try {
        const res = await fetch('https://restcountries.com/v3.1/all?fields=name,flags,cca2,capital,region,timezones');
        const data = await res.json();
        allCountries.value = data.sort((a, b) => a.name.common.localeCompare(b.name.common));
    } catch (err) {
        console.error('Failed to fetch countries:', err);
        triggerToast('Failed to load countries. Please check your internet connection.', 'error');
    } finally {
        isLoadingCountries.value = false;
    }
};

const openCountryModal = () => {
    showCountryModal.value = true;
    fetchCountries();
};

const addCountry = (country) => {
    // Attempt to guess Timezone from Capital + Region
    // This is a heuristic because the API returns offsets (UTC+01:00) or sometimes just the region
    let zone = null;


    // Check for duplicates
    const exists = locations.value.some(loc => loc.code === country.cca2.toLowerCase());
    if (exists) {
        triggerToast(`${country.name.common} is already in the list.`, 'warning');
        return;
    }

    // Check if it's the base country (Egypt or Nigeria)
    // Note: Base country is handled by the `baseCountry` ref, so we check name match too
    // Strategy 2: Check if Region/Capital is a valid IANA zone
    if (country.capital && country.capital.length > 0) {
        const capitalFormatted = country.capital[0].replace(/\s+/g, '_');
        const candidate1 = `${country.region}/${capitalFormatted}`;
        const candidate2 = `${country.region}/${country.capital[0]}`; // Sometimes spaces are valid or needed

        if (DateTime.now().setZone(candidate1).isValid) zone = candidate1;
        else if (DateTime.now().setZone(candidate2).isValid) zone = candidate2;
    }

    // Strategy 3: Fallback to a UTC offset if available (Less ideal for DST)
    if (!zone && country.timezones && country.timezones.length > 0) {
        // Luxon can handle 'UTC+5' etc.
        zone = country.timezones[0];
    }

    // Add to list if valid
    if (zone) {
        locations.value.push({
            label: country.name.common,
            zone: zone,
            flag: country.flags.svg,
            code: country.cca2.toLowerCase()
        });
        showCountryModal.value = false;
        searchQuery.value = '';
        triggerToast(`${country.name.common} added successfully!`, 'success');
    } else {
        triggerToast(`Could not determine a valid timezone for ${country.name.common}.`, 'error');
    }
};

const removeLocation = (index) => {
    // Prevent removing the first one if it's the base? 
    // Or just let user manage list freely except maybe the "System Time" or Base
    locations.value.splice(index, 1);
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

                <!-- Base Time Selector -->
                <div class="flex items-center gap-2">
                    <span class="text-sm font-bold text-slate-500 hidden md:inline">Base Time:</span>
                    <select v-model="baseCountry"
                        class="cursor-pointer bg-slate-50 border-2 border-slate-100 text-slate-700 text-sm font-bold rounded-xl px-3 py-2 hover:border-[#5B4DFF] focus:border-[#5B4DFF] outline-none transition-all">
                        <option value="Nigeria">Nigeria 🇳🇬</option>
                        <option value="Egypt">Egypt 🇪🇬</option>
                    </select>
                </div>
            </header>

            <!-- Context Info -->
            <div class="bg-blue-50 border border-blue-100 p-4 rounded-2xl flex items-start gap-3">
                <svg xmlns="http://www.w3.org/2000/svg" class="w-6 h-6 text-blue-500 flex-shrink-0 mt-0.5"
                    viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
                    <circle cx="12" cy="12" r="10" />
                    <path d="M12 16v-4" />
                    <path d="M12 8h.01" />
                </svg>
                <p class="text-sm text-blue-800 font-medium">
                    You are now entering the time based on
                    <span class="font-black underline decoration-2 underline-offset-2">{{ baseCountry }}</span>
                    time ({{ baseZone }}).
                </p>
            </div>

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
                                class="cursor-pointer text-slate-400 hover:text-[#5B4DFF] hover:bg-[#5B4DFF]/5 p-2 rounded-full transition-all"
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
                                    class="cursor-pointer py-2 rounded-xl font-mono text-sm font-bold border-2 transition-all"
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
                                    <img v-if="baseCountry === 'Nigeria'" src="https://flagcdn.com/w80/ng.png"
                                        crossorigin="anonymous" alt="Nigeria"
                                        class="w-8 h-8 rounded-full object-cover border" style="border-color: #f1f5f9;">
                                    <img v-else src="https://flagcdn.com/w80/eg.png" alt="Egypt" crossorigin="anonymous"
                                        class="w-8 h-8 rounded-full object-cover border" style="border-color: #f1f5f9;">
                                    {{ baseCountry }}
                                </h2>
                                <p class="font-medium text-sm mt-1 pl-10" style="color: #64748b;">
                                    {{ selectedDate.toFormat('cccc, LLL dd') }}
                                </p>
                            </div>
                            <div class="text-right flex flex-col items-end">
                                <!-- Action Buttons -->
                                <div class="flex gap-2 mb-2" data-html2canvas-ignore>
                                    <button @click.stop="copySchedule"
                                        class="cursor-pointer text-xs font-bold flex items-center gap-1 px-3 py-1.5 rounded-lg transition-colors border"
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
                                        class="cursor-pointer text-xs font-bold flex items-center gap-1 px-3 py-1.5 rounded-lg transition-colors border"
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
                                    Time ({{ baseCountry }})</span>
                            </div>
                        </div>

                        <!-- Results Grid (User List Style) -->
                        <div class="grid grid-cols-1 md:grid-cols-2 gap-4">
                            <div v-for="(item, idx) in scheduledTimes.slice(1)" :key="item.zone + idx"
                                class="relative flex items-center p-4 rounded-3xl border transition-all duration-300 group hover:shadow-lg"
                                style="background-color: #ffffff; border-color: #f1f5f9;">
                                <!-- Avatar -->
                                <div class="w-12 h-12 rounded-full flex items-center justify-center flex-shrink-0 border overflow-hidden"
                                    style="background-color: #f8fafc; border-color: #f1f5f9;">
                                    <img :src="`https://flagcdn.com/w80/${item.code}.png`" :alt="item.label"
                                        crossorigin="anonymous" class="w-full h-full object-cover opacity-90" />
                                </div>

                                <!-- Middle -->
                                <div class="ml-4 flex-1 min-w-0">
                                    <h3 class="text-sm font-bold truncate pr-6" style="color: #0f172a;">{{ item.label }}
                                    </h3>
                                    <p class="text-xs font-medium truncate pb-1" style="color: #94a3b8;">{{ item.region
                                        }}
                                    </p>
                                </div>
                                <button @click.stop="removeLocation(idx + 1)"
                                    class="cursor-pointer absolute -top-2 -right-2 bg-red-100 text-red-500 rounded-full p-1 opacity-0 group-hover:opacity-100 transition-opacity hover:bg-red-200 shadow-sm"
                                    title="Remove" style="background-color: #fee2e2; color: #ef4444;">
                                    <svg xmlns="http://www.w3.org/2000/svg" width="14" height="14" viewBox="0 0 24 24"
                                        fill="none" stroke="currentColor" stroke-width="3" stroke-linecap="round"
                                        stroke-linejoin="round">
                                        <line x1="18" y1="6" x2="6" y2="18"></line>
                                        <line x1="6" y1="6" x2="18" y2="18"></line>
                                    </svg>
                                </button>

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

                        <!-- Add Country Button (Full Width) -->
                        <button @click="openCountryModal"
                            class="cursor-pointer w-full py-4 mt-4 rounded-3xl border-2 border-dashed text-slate-500 hover:border-[#5B4DFF] hover:text-[#5B4DFF] hover:bg-[#5B4DFF]/5 transition-all flex items-center justify-center gap-2 font-bold"
                            style="color: #64748b; border-color: #cbd5e1;">
                            <svg xmlns="http://www.w3.org/2000/svg" width="20" height="20" viewBox="0 0 24 24"
                                fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round"
                                stroke-linejoin="round">
                                <line x1="12" y1="5" x2="12" y2="19"></line>
                                <line x1="5" y1="12" x2="19" y2="12"></line>
                            </svg>
                            Add Another Country
                        </button>

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

                <!-- Country Selection Modal -->
                <div v-if="showCountryModal"
                    class="fixed inset-0 z-50 flex items-center justify-center p-4 bg-slate-900/20 backdrop-blur-sm"
                    @click.self="showCountryModal = false">
                    <div
                        class="bg-white rounded-[2rem] p-6 shadow-2xl max-w-lg w-full h-[80vh] flex flex-col animate-in fade-in zoom-in duration-200">
                        <div class="flex items-center justify-between mb-4">
                            <h3 class="text-xl font-bold text-slate-900">Add Country</h3>
                            <button @click="showCountryModal = false" class="text-slate-400 hover:text-slate-600">
                                <svg xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24"
                                    fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round"
                                    stroke-linejoin="round">
                                    <line x1="18" y1="6" x2="6" y2="18"></line>
                                    <line x1="6" y1="6" x2="18" y2="18"></line>
                                </svg>
                            </button>
                        </div>

                        <!-- Search -->
                        <div class="relative mb-4">
                            <input v-model="searchQuery" type="text" placeholder="Search countries..."
                                class="w-full bg-slate-50 border border-slate-200 rounded-xl py-3 px-4 pl-10 font-bold text-slate-700 focus:outline-none focus:border-[#5B4DFF] focus:ring-1 focus:ring-[#5B4DFF]" />
                            <svg class="absolute left-3 top-3.5 text-slate-400" xmlns="http://www.w3.org/2000/svg"
                                width="18" height="18" viewBox="0 0 24 24" fill="none" stroke="currentColor"
                                stroke-width="2">
                                <circle cx="11" cy="11" r="8"></circle>
                                <line x1="21" y1="21" x2="16.65" y2="16.65"></line>
                            </svg>
                        </div>

                        <!-- List -->
                        <div class="flex-1 overflow-y-auto no-scrollbar space-y-2">
                            <div v-if="isLoadingCountries" class="flex justify-center py-8 text-[#5B4DFF]">
                                <svg class="animate-spin h-8 w-8" xmlns="http://www.w3.org/2000/svg" fill="none"
                                    viewBox="0 0 24 24">
                                    <circle class="opacity-25" cx="12" cy="12" r="10" stroke="currentColor"
                                        stroke-width="4"></circle>
                                    <path class="opacity-75" fill="currentColor"
                                        d="M4 12a8 8 0 018-8V0C5.373 0 0 5.373 0 12h4zm2 5.291A7.962 7.962 0 014 12H0c0 3.042 1.135 5.824 3 7.938l3-2.647z">
                                    </path>
                                </svg>
                            </div>

                            <button v-for="country in filteredCountries" :key="country.cca2"
                                @click="addCountry(country)"
                                class="w-full flex items-center gap-3 p-3 rounded-xl hover:bg-slate-50 transition-colors text-left group">
                                <div
                                    class="w-10 h-10 rounded-full border border-slate-100 overflow-hidden bg-white flex-shrink-0">
                                    <img :src="country.flags.svg" :alt="country.name.common" crossorigin="anonymous"
                                        class="w-full h-full object-cover" />
                                </div>
                                <div>
                                    <h4 class="text-sm font-bold text-slate-800 group-hover:text-[#5B4DFF]">{{
                                        country.name.common }}</h4>
                                    <p class="text-xs text-slate-500">{{ country.region }}</p>
                                </div>
                            </button>

                            <div v-if="!isLoadingCountries && filteredCountries.length === 0"
                                class="text-center py-8 text-slate-400">
                                <p>No countries found.</p>
                            </div>
                        </div>
                    </div>
                </div>

                <!-- Toast Notification -->
                <Transition enter-active-class="transition ease-out duration-300"
                    enter-from-class="transform translate-y-8 opacity-0"
                    enter-to-class="transform translate-y-0 opacity-100"
                    leave-active-class="transition ease-in duration-200"
                    leave-from-class="transform translate-y-0 opacity-100"
                    leave-to-class="transform translate-y-8 opacity-0">
                    <div v-if="toast.show"
                        class="fixed bottom-8 left-1/2 -translate-x-1/2 z-[100] px-6 py-3 rounded-xl shadow-2xl flex items-center gap-3 font-bold text-sm min-w-[300px]"
                        :class="{
                            'bg-slate-900 text-white': toast.type === 'success',
                            'bg-red-500 text-white': toast.type === 'error',
                            'bg-orange-500 text-white': toast.type === 'warning'
                        }">
                        <svg v-if="toast.type === 'success'" xmlns="http://www.w3.org/2000/svg" width="20" height="20"
                            viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"
                            stroke-linecap="round" stroke-linejoin="round">
                            <path d="M22 11.08V12a10 10 0 1 1-5.93-9.14"></path>
                            <polyline points="22 4 12 14.01 9 11.01"></polyline>
                        </svg>
                        <svg v-if="toast.type === 'error'" xmlns="http://www.w3.org/2000/svg" width="20" height="20"
                            viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"
                            stroke-linecap="round" stroke-linejoin="round">
                            <circle cx="12" cy="12" r="10"></circle>
                            <line x1="15" y1="9" x2="9" y2="15"></line>
                            <line x1="9" y1="9" x2="15" y2="15"></line>
                        </svg>
                        <svg v-if="toast.type === 'warning'" xmlns="http://www.w3.org/2000/svg" width="20" height="20"
                            viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"
                            stroke-linecap="round" stroke-linejoin="round">
                            <path
                                d="M10.29 3.86L1.82 18a2 2 0 0 0 1.71 3h16.94a2 2 0 0 0 1.71-3L13.71 3.86a2 2 0 0 0-3.42 0z">
                            </path>
                            <line x1="12" y1="9" x2="12" y2="13"></line>
                            <line x1="12" y1="17" x2="12.01" y2="17"></line>
                        </svg>
                        {{ toast.message }}
                    </div>
                </Transition>

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
