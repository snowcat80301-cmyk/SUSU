import React, { useState, useEffect, useCallback, useMemo } from 'react';
import { 
  LayoutDashboard, 
  Users, 
  CalendarCheck, 
  Calculator, 
  FileText, 
  BookOpen, 
  Settings, 
  Search, 
  FileDown, 
  Pencil, 
  Trash2, 
  Phone, 
  MapPin, 
  Navigation, 
  NotebookPen, 
  X, 
  FileSpreadsheet, 
  FileCode, 
  Sparkles, 
  Save, 
  Plus, 
  ChevronRight, 
  ChevronDown, 
  CloudUpload, 
  Sun, 
  Minus, 
  CheckCircle, 
  BriefcaseMedical,
  Loader2,
  Cloud,
  CloudOff,
  Link as LinkIcon,
  Wifi,
  WifiOff
} from 'lucide-react';

// Firebase Imports
import { initializeApp } from 'firebase/app';
import { 
  getAuth, 
  signInAnonymously, 
  onAuthStateChanged, 
  User, 
  signInWithCustomToken 
} from 'firebase/auth';
import { 
  getFirestore, 
  collection, 
  doc, 
  setDoc, 
  deleteDoc, 
  onSnapshot, 
  query, 
  writeBatch 
} from 'firebase/firestore';

// --- Firebase Initialization ---
const firebaseConfig = {
    apiKey: "AIzaSyCZGIyjY2l1gaarCXW6gJ2Nk-6sYoi9DOE",
    authDomain: "lc-ai-28847.firebaseapp.com",
    projectId: "lc-ai-28847",
    storageBucket: "lc-ai-28847.firebasestorage.app",
    messagingSenderId: "284514158464",
    appId: "1:284514158464:web:5db0372d5754c3c4e52365"
  };

const appId = "care-helper-v1"; //

let auth: any;
let db: any;
if (Object.keys(firebaseConfig).length > 0) {
  const app = initializeApp(firebaseConfig);
  auth = getAuth(app);
  db = getFirestore(app);
}

// --- Types ---

export enum ClientStatus {
  ACTIVE = '服務中',
  PAUSED = '暫停',
  CLOSED = '結案',
  HOSPITALIZED = '住院'
}

export type WelfareType = 'normal' | 'midLow' | 'low';

export interface Client {
  id: string;
  name: string;
  gender: '男性' | '女性' | '未知';
  age?: string;
  welfare: string;
  phone: string;
  address: string;
  status: ClientStatus;
  freq_type: string;
  supervisor?: string;
  last_visit_date?: string;
}

export interface VisitRecord {
  id: string;
  clientId: string;
  date: string;
  displayDate: string;
  method: string;
  interviewee: string;
  purpose: string;
  health: string;
  satisfaction: string;
  special: string;
  conclusion: string;
  recommendation: string;
}

export interface ManualTimelineEntry {
  clientId: string;
  year: number;
  month: number;
  type: 'home' | 'phone' | 'other' | null;
}

export interface ServiceItem {
  code: string;
  name: string;
  category: string;
  pay: number;
  cost: { normal: number; midLow: number; low: number; };
}

export interface SOPItem {
  id: string;
  title: string;
  desc: string;
  category: 'routine' | 'govt' | 'compal';
}

export interface AppConfig {
  apiKey: string;
  gasUrl: string;
  syncId: string; // New: For manual group syncing
}

export type ViewType = 'home' | 'clients' | 'timeline' | 'record' | 'sop' | 'calculator';

// --- Constants ---

export const SERVICE_ITEMS: ServiceItem[] = [
  { code: 'BA01', category: '基本照顧', name: '基本身體清潔', pay: 260, cost: { normal: 41, midLow: 13, low: 0 } },
  { code: 'BA02', category: '基本照顧', name: '基本日常照顧', pay: 195, cost: { normal: 31, midLow: 9, low: 0 } },
  { code: 'BA03', category: '基本照顧', name: '測量生命徵象', pay: 35, cost: { normal: 5, midLow: 1, low: 0 } },
  { code: 'BA04', category: '基本照顧', name: '協助進食或管灌餵食', pay: 130, cost: { normal: 20, midLow: 6, low: 0 } },
  { code: 'BA05', category: '基本照顧', name: '餐食照顧', pay: 310, cost: { normal: 49, midLow: 15, low: 0 } },
  { code: 'BA07', category: '身體活動', name: '協助沐浴及洗頭', pay: 325, cost: { normal: 52, midLow: 16, low: 0 } },
  { code: 'BA08', category: '身體活動', name: '足部照護', pay: 500, cost: { normal: 80, midLow: 25, low: 0 } },
  { code: 'BA10', category: '身體活動', name: '翻身拍背', pay: 155, cost: { normal: 24, midLow: 7, low: 0 } },
  { code: 'BA11', category: '身體活動', name: '肢體關節活動', pay: 195, cost: { normal: 31, midLow: 9, low: 0 } },
  { code: 'BA12', category: '身體活動', name: '協助上下樓梯', pay: 130, cost: { normal: 20, midLow: 6, low: 0 } },
  { code: 'BA24', category: '身體活動', name: '協助排泄', pay: 220, cost: { normal: 35, midLow: 11, low: 0 } },
  { code: 'BA13', category: '外出陪同', name: '陪同外出', pay: 195, cost: { normal: 31, midLow: 9, low: 0 } },
  { code: 'BA14', category: '外出陪同', name: '陪同就醫', pay: 685, cost: { normal: 109, midLow: 34, low: 0 } },
  { code: 'BA15', category: '外出陪同', name: '家務協助', pay: 195, cost: { normal: 31, midLow: 9, low: 0 } },
  { code: 'BA16', category: '外出陪同', name: '代購／代領／代送', pay: 130, cost: { normal: 20, midLow: 6, low: 0 } },
  { code: 'BA17a', category: '醫療輔助', name: '人工氣道抽吸', pay: 75, cost: { normal: 12, midLow: 3, low: 0 } },
  { code: 'BA17b', category: '醫療輔助', name: '口鼻抽吸', pay: 65, cost: { normal: 10, midLow: 3, low: 0 } },
  { code: 'BA17c', category: '醫療輔助', name: '管路清潔', pay: 50, cost: { normal: 8, midLow: 2, low: 0 } },
  { code: 'BA17d', category: '醫療輔助', name: '甘油球／血糖檢測', pay: 50, cost: { normal: 8, midLow: 2, low: 0 } },
  { code: 'BA17e', category: '醫療輔助', name: '依指示置入藥盒', pay: 50, cost: { normal: 8, midLow: 2, low: 0 } },
  { code: 'BA18', category: '看視陪伴', name: '安全看視', pay: 200, cost: { normal: 32, midLow: 10, low: 0 } },
  { code: 'BA20', category: '看視陪伴', name: '陪伴服務', pay: 175, cost: { normal: 28, midLow: 8, low: 0 } },
  { code: 'BA22', category: '看視陪伴', name: '巡視服務', pay: 130, cost: { normal: 20, midLow: 6, low: 0 } },
  { code: 'BA23', category: '看視陪伴', name: '協助洗頭', pay: 200, cost: { normal: 32, midLow: 10, low: 0 } },
];

// --- Services ---

const mapRowToClient = (row: any, headers: string[]): Client | null => {
  const findIdx = (keywords: string[]) => headers.findIndex(h => h && keywords.some(k => h.includes(k)));
  
  const idxName = findIdx(['姓名']);
  const idxGender = findIdx(['性別']);
  const idxAge = findIdx(['年齡']);
  const idxPhone = findIdx(['電話', '手機']);
  const idxWelfare = findIdx(['福利', '身分別']);
  const idxAddress = findIdx(['個案居住地址', '居住地址', '地址']);

  if (idxName === -1) return null;

  const getValue = (idx: number) => {
    if (Array.isArray(row)) return row[idx];
    return row[headers[idx]];
  };

  const rawName = getValue(idxName);
  if (!rawName || (typeof rawName === 'string' && rawName.trim() === '姓名')) return null;

  const rawWelfare = String(getValue(idxWelfare) || '');
  let finalWelfare = '一般戶';
  if (rawWelfare.includes('第一類') || rawWelfare.includes('低收')) finalWelfare = '低收入';
  else if (rawWelfare.includes('第二類') || rawWelfare.includes('中低')) finalWelfare = '中低收';
  else if (rawWelfare.includes('第三類')) finalWelfare = '一般戶';

  let address = '未提供';
  if (idxAddress !== -1) {
    address = String(getValue(idxAddress) || '').replace(/"/g, '').trim();
  }

  return {
    id: `imp-${Date.now()}-${Math.random().toString(36).substr(2, 9)}`,
    name: String(rawName).trim(),
    gender: (String(getValue(idxGender) || '').replace(/"/g, '').trim()) as any || '未知',
    age: String(getValue(idxAge) || '').replace(/"/g, '').trim(),
    welfare: finalWelfare,
    phone: String(getValue(idxPhone) || '').replace(/"/g, '').trim() || '未提供',
    address: address,
    status: ClientStatus.ACTIVE,
    freq_type: '電電家',
    supervisor: '匯入資料'
  };
};

const parseCsvData = (text: string): Client[] => {
  const lines = text.split(/\r\n|\n|\r/);
  const clients: Client[] = [];
  
  let headerIndex = -1;
  for (let i = 0; i < Math.min(lines.length, 30); i++) {
    if (lines[i].includes('姓名') && (lines[i].includes('性別') || lines[i].includes('福利'))) {
      headerIndex = i;
      break;
    }
  }

  if (headerIndex === -1) return [];

  const headers = lines[headerIndex].split(',').map(h => h.trim());
  
  for (let i = headerIndex + 1; i < lines.length; i++) {
    const line = lines[i].trim();
    if (!line) continue;
    const row = line.split(',');
    const client = mapRowToClient(row, headers);
    if (client) clients.push(client);
  }
  return clients;
};

const parseSheetData = (jsonData: any[][]): Client[] => {
  let headerIndex = -1;
  for (let i = 0; i < Math.min(jsonData.length, 30); i++) {
    const rowStr = JSON.stringify(jsonData[i]);
    if (rowStr.includes('姓名') && (rowStr.includes('性別') || rowStr.includes('福利'))) {
      headerIndex = i;
      break;
    }
  }

  if (headerIndex === -1) return [];

  const headers = jsonData[headerIndex].map(h => String(h).trim());
  const clients: Client[] = [];

  for (let i = headerIndex + 1; i < jsonData.length; i++) {
    const client = mapRowToClient(jsonData[i], headers);
    if (client) clients.push(client);
  }
  return clients;
};

const parseFile = (file: File): Promise<any[]> => {
  return new Promise((resolve) => {
    const reader = new FileReader();
    reader.onload = async (e) => {
      const buffer = e.target?.result as ArrayBuffer;
      let clients: Client[] = [];
      try {
        const decoder = new TextDecoder('big5');
        const text = decoder.decode(buffer);
        if (text.includes('姓名') && text.includes('性別')) {
          clients = parseCsvData(text);
          if (clients.length > 0) { resolve(clients); return; }
        }
      } catch (err) {}
      try {
        const decoder = new TextDecoder('utf-8');
        const text = decoder.decode(buffer);
        if (text.includes('姓名') && text.includes('性別')) {
          clients = parseCsvData(text);
          if (clients.length > 0) { resolve(clients); return; }
        }
      } catch (err) {}
      if ((window as any).XLSX) {
        try {
          const workbook = (window as any).XLSX.read(buffer, { type: 'array' });
          const sheetName = workbook.SheetNames[0];
          const sheet = workbook.Sheets[sheetName];
          const json = (window as any).XLSX.utils.sheet_to_json(sheet, { header: 1, defval: '' });
          clients = parseSheetData(json);
          if (clients.length > 0) { resolve(clients); return; }
        } catch (err) {}
      }
      resolve([]);
    };
    reader.readAsArrayBuffer(file);
  });
};

const parseHtmlReport = (htmlText: string): Partial<Client> => {
  const parser = new DOMParser();
  const doc = parser.parseFromString(htmlText, 'text/html');
  const bodyText = doc.body.innerText.replace(/\s+/g, ' ');
  const nameIdMatch = bodyText.match(/個案姓名[\/\s]*身分證字號\s*[:：]\s*([^\/]+)\/([A-Z][1289]\d{8})/);
  const phoneMatch = bodyText.match(/(?:電話|手機)\s*[:：]?\s*([\d\-\(\)\s]{9,15})/);
  const addrMatch = bodyText.match(/居住[\(\s]*通訊[\)\s]*地\s*址\s*[:：]?\s*([^\s]{5,}(?:縣|市)[^\s]{5,})/);
  return {
    id: nameIdMatch ? nameIdMatch[2] : `html-${Date.now()}`,
    name: nameIdMatch ? nameIdMatch[1].trim() : '未知',
    phone: phoneMatch ? phoneMatch[1].trim().replace(/\s/g, '') : '未提供',
    address: addrMatch ? addrMatch[1].trim() : '未提供',
    status: ClientStatus.ACTIVE,
    freq_type: '電電家'
  };
};

async function generateVisitRecord(apiKey: string, data: {
  clientName: string,
  gender: string,
  welfare: string,
  method: string,
  interviewee: string,
  purpose: string,
  recentActivities?: string
}) {
  const prompt = `你是一位擁有 10 年經驗的「長照居家服務督導員」。請根據以下資訊撰寫訪視紀錄。
  
  【個案】${data.clientName} (${data.gender}, ${data.welfare})
  【本次訪視】方式：${data.method} / 受訪者：${data.interviewee}
  【原始目的】${data.purpose}
  【觀察筆記】${data.recentActivities || '無特別備註'}

  【格式要求 - 請嚴格遵守】
  請回傳 JSON，包含以下欄位內容：
  1. purpose: 訪視目的，請使用數字編號 (1. 2.) 條列。
  2. health: 健康狀況，請使用項目符號 (*) 條列。
  3. satisfaction: 滿意度評估，請使用項目符號 (*) 條列。
  4. special: 特殊狀況或提醒，請使用項目符號 (*) 條列，若無則寫「無特殊異狀」。
  5. conclusion: 結論與建議，請使用數字編號 (1. 2.) 條列。
  `;

  try {
    const response = await fetch(`https://generativelanguage.googleapis.com/v1beta/models/gemini-2.5-flash-preview-09-2025:generateContent?key=${apiKey}`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        contents: [{ parts: [{ text: prompt }] }],
        generationConfig: {
          responseMimeType: "application/json",
          responseSchema: {
            type: "OBJECT",
            properties: {
              purpose: { type: "STRING" },
              health: { type: "STRING" },
              satisfaction: { type: "STRING" },
              special: { type: "STRING" },
              conclusion: { type: "STRING" }
            }
          }
        }
      })
    });
    if (!response.ok) throw new Error(await response.text());
    const result = await response.json();
    return JSON.parse(result.candidates?.[0]?.content?.parts?.[0]?.text || '{}');
  } catch (error) {
    console.error("Gemini API Error:", error);
    throw error;
  }
}

// --- Components ---

const SettingsModal: React.FC<{ config: AppConfig; onClose: () => void; onSave: (config: AppConfig) => void; onUploadLocal: () => void }> = ({ config, onClose, onSave, onUploadLocal }) => {
  const [localKey, setLocalKey] = useState(config.apiKey || '');
  const [localUrl, setLocalUrl] = useState(config.gasUrl || '');
  const [localSyncId, setLocalSyncId] = useState(config.syncId || '');

  return (
    <div className="fixed inset-0 bg-black/60 backdrop-blur-sm z-[100] flex items-center justify-center p-6">
      <div className="bg-white w-full max-w-sm rounded-2xl shadow-2xl overflow-hidden">
        <div className="p-4 bg-[#fcf9f2] border-b border-[#efebe9] flex justify-between items-center">
          <h3 className="font-bold text-[#5d4037]">系統設定</h3>
          <button onClick={onClose} className="text-gray-400 hover:text-gray-600"><X size={20} /></button>
        </div>
        <div className="p-6 space-y-5">
          <div className="space-y-2 bg-blue-50 p-3 rounded-xl border border-blue-100">
             <label className="block text-xs font-bold text-blue-800 flex items-center gap-1"><LinkIcon size={12}/> 跨裝置同步代碼 (Sync ID)</label>
             <input 
              type="text" 
              value={localSyncId}
              onChange={(e) => setLocalSyncId(e.target.value)}
              placeholder="例如：NurseLin888"
              className="w-full p-2 text-sm border border-blue-200 rounded-lg bg-white focus:ring-2 focus:ring-blue-400 outline-none"
             />
             <p className="text-[10px] text-blue-600">
               在所有裝置輸入相同的代碼，資料即可互通。<br/>
               <span className="font-bold">注意：</span>知道此代碼的人皆可存取資料，請設定複雜一點。
             </p>
          </div>

          <div className="space-y-2">
            <label className="block text-xs font-bold text-gray-700">Google Gemini API Key</label>
            <input 
              type="password" 
              value={localKey}
              onChange={(e) => setLocalKey(e.target.value)}
              placeholder="請輸入 API Key"
              className="w-full p-3 text-sm border border-[#d7ccc8] rounded-xl bg-gray-50 focus:bg-white focus:ring-2 focus:ring-[#8d6e63] outline-none transition"
            />
          </div>
          <div className="space-y-2">
            <label className="block text-xs font-bold text-gray-700">GAS Webhook URL</label>
            <input 
              type="text" 
              value={localUrl}
              onChange={(e) => setLocalUrl(e.target.value)}
              placeholder="Google Apps Script URL"
              className="w-full p-3 text-sm border border-[#d7ccc8] rounded-xl bg-gray-50 focus:bg-white focus:ring-2 focus:ring-[#8d6e63] outline-none transition"
            />
          </div>
          
          <div className="pt-2 border-t border-dashed border-gray-200">
             <button onClick={onUploadLocal} className="w-full py-2 text-xs font-bold text-[#8d6e63] border border-[#d7ccc8] rounded-lg hover:bg-[#fcf9f2] flex items-center justify-center gap-1">
               <CloudUpload size={12} /> 將本機資料上傳至此代碼群組
             </button>
             <p className="text-[9px] text-gray-400 text-center mt-1">換新裝置或剛設定同步代碼時使用</p>
          </div>

          <div className="flex gap-3 pt-2">
            <button onClick={onClose} className="flex-1 py-3 text-sm font-bold text-gray-500 bg-gray-100 rounded-xl hover:bg-gray-200 transition">取消</button>
            <button onClick={() => onSave({ apiKey: localKey, gasUrl: localUrl, syncId: localSyncId })} className="flex-1 py-3 text-sm font-bold text-white bg-[#8d6e63] rounded-xl hover:bg-[#5d4037] shadow-lg shadow-[#8d6e63]/30 transition">儲存設定</button>
          </div>
        </div>
      </div>
    </div>
  );
};

const Dashboard: React.FC<{ stats: { active: number; monthlyVisits: number }; gasUrl: string; clients: Client[]; records: Record<string, VisitRecord[]>; syncId: string }> = ({ stats, gasUrl, clients, records, syncId }) => {
  const handleSync = async () => {
    if (!gasUrl) { alert('請先設定 GAS Webhook URL'); return; }
    try {
      await fetch(gasUrl, { method: 'POST', mode: 'no-cors', body: JSON.stringify({ clients, records }) });
      alert('備份請求已發送 (Sheet)');
    } catch (e) { alert('同步失敗'); }
  };

  return (
    <div className="space-y-4 animate-in fade-in duration-300">
      <div className="bg-gradient-to-br from-[#8d6e63] to-[#a1887f] text-white p-6 rounded-2xl shadow-lg relative overflow-hidden">
        <div className="relative z-10">
          <h2 className="text-2xl font-bold mb-1">早安，督導！</h2>
          {syncId ? (
             <div className="flex items-center gap-2 text-[10px] font-bold bg-green-500/30 w-fit px-2 py-1 rounded mb-2 border border-green-400/30">
                <LinkIcon size={10} /> 已連線同步群組：{syncId}
             </div>
          ) : (
             <div className="flex items-center gap-2 text-[10px] font-bold bg-white/20 w-fit px-2 py-1 rounded mb-2">
                <CloudOff size={10} /> 單機私密模式 (未同步)
             </div>
          )}
          
          <button onClick={handleSync} className="mt-4 bg-white/20 hover:bg-white/30 text-white text-xs font-bold py-2 px-4 rounded-lg flex items-center gap-2 backdrop-blur-sm transition border border-white/30">
            <CloudUpload size={16} /> 匯出至 Google Sheets
          </button>
        </div>
        <Sun className="absolute right-[-10px] top-[-10px] text-white opacity-10" size={120} />
      </div>
      <div className="grid grid-cols-2 gap-4">
        <div className="bg-white p-4 rounded-xl shadow-sm border border-[#efebe9] flex flex-col items-center justify-center gap-2">
          <span className="text-xs text-gray-400">目前服務個案</span>
          <span className="text-2xl font-bold text-[#8d6e63]">{stats.active}</span>
        </div>
        <div className="bg-white p-4 rounded-xl shadow-sm border border-[#efebe9] flex flex-col items-center justify-center gap-2">
          <span className="text-xs text-gray-400">本月已訪視</span>
          <span className="text-2xl font-bold text-[#5d4037]">{stats.monthlyVisits}</span>
        </div>
      </div>
    </div>
  );
};

const ClientList: React.FC<{ clients: Client[]; onDeleteClient: (id: string) => void; onWriteRecord: (id: string) => void; onImport: () => void; onEditClient: (client: Client) => void }> = ({ clients, onDeleteClient, onWriteRecord, onImport, onEditClient }) => {
  const [searchTerm, setSearchTerm] = useState('');
  const [filter, setFilter] = useState<ClientStatus | 'ALL'>('ALL');

  const filteredClients = clients.filter(c => {
    const searchStr = (c.name + c.phone + c.address).toLowerCase();
    const matchesSearch = searchStr.includes(searchTerm.toLowerCase());
    const matchesFilter = filter === 'ALL' || c.status === filter;
    return matchesSearch && matchesFilter;
  });

  return (
    <div className="space-y-4">
      <div className="flex justify-between items-center bg-white p-3 rounded-xl shadow-sm border border-[#efebe9] sticky top-0 z-10">
        <div className="flex-1 mr-2 relative">
          <Search className="absolute left-3 top-2.5 text-gray-400" size={16} />
          <input type="text" placeholder="搜尋姓名、電話或地址..." value={searchTerm} onChange={(e) => setSearchTerm(e.target.value)} className="w-full pl-9 pr-2 py-2 bg-[#fcf9f2] border border-[#d7ccc8] rounded-lg text-sm focus:outline-none focus:ring-2 focus:ring-[#8d6e63]" />
        </div>
        <button onClick={onImport} className="flex items-center gap-1 bg-[#8d6e63] text-white px-3 py-2 rounded-lg text-xs font-bold shadow hover:opacity-90 transition active:scale-95">
          <FileDown size={14} /> 匯入
        </button>
      </div>
      <div className="flex gap-2 text-xs overflow-x-auto no-scrollbar pb-1">
        {['ALL', ...Object.values(ClientStatus)].map(f => (
          <button key={f} onClick={() => setFilter(f as any)} className={`px-3 py-1.5 rounded-full font-bold whitespace-nowrap border transition-all ${filter === f ? 'bg-[#d7ccc8] text-[#5d4037] border-[#d7ccc8]' : 'bg-white text-gray-500 border-[#d7ccc8] hover:bg-gray-50'}`}>
            {f === 'ALL' ? '全部' : f}
          </button>
        ))}
      </div>
      <div className="grid grid-cols-1 gap-3 min-h-[300px]">
        {filteredClients.length === 0 ? (
          <div className="text-center py-10 text-gray-400 text-sm italic">尚無符合條件的個案</div>
        ) : (
          filteredClients.map(c => (
            <div key={c.id} className={`p-4 rounded-xl border border-[#efebe9] bg-white shadow-sm hover:shadow-md transition-all duration-300 ${c.status === ClientStatus.CLOSED ? 'opacity-60 bg-gray-50' : ''}`}>
              <div className="flex justify-between items-start mb-2">
                <div onClick={() => onEditClient(c)} className="cursor-pointer">
                  <h3 className="font-bold text-[#4e342e] text-base flex items-center gap-2 group">
                    {c.name} 
                    <span className="text-[10px] font-normal text-gray-400 px-1.5 py-0.5 bg-gray-100 rounded">{c.gender}</span>
                    {c.age && <span className="text-[10px] text-gray-400 ml-1">({c.age}歲)</span>}
                    <Pencil size={12} className="opacity-0 group-hover:opacity-100 text-[#8d6e63] transition-opacity" />
                  </h3>
                  <p className="text-[10px] text-gray-400 mt-1">{c.welfare} • {c.supervisor || '未指定'}</p>
                </div>
                <span className={`text-[10px] px-2 py-0.5 rounded-full font-bold ${c.status === ClientStatus.ACTIVE ? 'bg-green-100 text-green-700' : c.status === ClientStatus.CLOSED ? 'bg-gray-100 text-gray-600' : 'bg-orange-100 text-orange-700'}`}>
                  {c.status}
                </span>
              </div>
              <div className="space-y-1.5 text-xs text-gray-600 mb-4 bg-[#fcf9f2] p-2 rounded-lg">
                <div className="flex items-center gap-2"><Phone size={12} className="text-[#8d6e63]" /> <a href={`tel:${c.phone}`} className="hover:underline font-medium">{c.phone}</a></div>
                <div className="flex items-center gap-2 cursor-pointer group" onClick={() => { if(c.address) window.open(`https://www.google.com/maps/search/?api=1&query=${encodeURIComponent(c.address)}`, '_blank')}}>
                  <MapPin size={12} className="text-[#8d6e63]" /> 
                  <span className="group-hover:text-[#8d6e63] flex-1 line-clamp-1 transition-colors">{c.address}</span>
                  <Navigation size={12} className="text-[#8d6e63] opacity-0 group-hover:opacity-100 transition-opacity" />
                </div>
              </div>
              <div className="flex gap-2">
                <button onClick={() => onWriteRecord(c.id)} className="flex-1 py-2 bg-[#8d6e63] text-white rounded-lg text-xs font-bold shadow-sm active:scale-95 transition flex items-center justify-center gap-2">
                  <NotebookPen size={14} /> 寫紀錄
                </button>
                <button onClick={() => onEditClient(c)} className="px-3 py-2 border border-[#d7ccc8] text-[#8d6e63] rounded-lg hover:bg-[#fcf9f2] active:scale-95 transition">
                  <Pencil size={14} />
                </button>
                <button onClick={() => onDeleteClient(c.id)} className="px-3 py-2 border border-red-100 text-red-400 rounded-lg hover:bg-red-50 active:scale-95 transition">
                  <Trash2 size={14} />
                </button>
              </div>
            </div>
          ))
        )}
      </div>
    </div>
  );
};

const RecordForm: React.FC<{ clients: Client[]; initialClientId: string | null; editRecord: VisitRecord | null; apiKey: string; onSave: (record: VisitRecord) => void; onCancel?: () => void }> = ({ clients, initialClientId, editRecord, apiKey, onSave, onCancel }) => {
  const [clientId, setClientId] = useState(initialClientId || '');
  const [date, setDate] = useState(new Date().toISOString().split('T')[0]);
  const [method, setMethod] = useState('家庭訪視');
  const [interviewee, setInterviewee] = useState('案主');
  const [purpose, setPurpose] = useState('');
  const [recentActivities, setRecentActivities] = useState('');
  const [health, setHealth] = useState('');
  const [satisfaction, setSatisfaction] = useState('');
  const [special, setSpecial] = useState('');
  const [conclusion, setConclusion] = useState('');
  const [recommendation, setRecommendation] = useState('');
  const [isGenerating, setIsGenerating] = useState(false);

  const selectedClient = clients.find(c => c.id === clientId);

  useEffect(() => {
    if (editRecord) {
      setClientId(editRecord.clientId);
      setDate(editRecord.date);
      setMethod(editRecord.method);
      setInterviewee(editRecord.interviewee);
      setPurpose(editRecord.purpose);
      setHealth(editRecord.health);
      setSatisfaction(editRecord.satisfaction);
      setSpecial(editRecord.special);
      setConclusion(editRecord.conclusion);
      setRecommendation(editRecord.recommendation || '');
    } else if (initialClientId) {
      setClientId(initialClientId);
      setPurpose(''); setRecentActivities(''); setHealth(''); setSatisfaction(''); setSpecial(''); setConclusion(''); setRecommendation('');
    }
  }, [editRecord, initialClientId]);

  const handleAI = async () => {
    if (!apiKey) { alert('請先在設定中填寫 Gemini API Key'); return; }
    if (!clientId || !purpose) { alert('請選擇個案並填寫簡要目的'); return; }

    setIsGenerating(true);
    try {
      const result = await generateVisitRecord(apiKey, {
        clientName: selectedClient?.name || '個案',
        gender: selectedClient?.gender || '未知',
        welfare: selectedClient?.welfare || '一般戶',
        method, interviewee, purpose, recentActivities
      });
      setPurpose(result.purpose || purpose);
      setHealth(result.health || '');
      setSatisfaction(result.satisfaction || '');
      setSpecial(result.special || '');
      setConclusion(result.conclusion || '');
    } catch (e) {
      alert('AI 生成失敗，請檢查網路或 API Key');
    } finally {
      setIsGenerating(false);
    }
  };

  const handleSave = () => {
    if (!clientId) { alert('請選擇個案'); return; }
    const d = new Date(date);
    const displayDate = `${d.getFullYear() - 1911}年${d.getMonth() + 1}月${d.getDate()}日`;
    const record: VisitRecord = {
      id: editRecord ? editRecord.id : Date.now().toString(),
      clientId, date, displayDate, method, interviewee, purpose, health, satisfaction, special, conclusion, recommendation
    };
    onSave(record);
    if (!editRecord) { setPurpose(''); setRecentActivities(''); setHealth(''); setSatisfaction(''); setSpecial(''); setConclusion(''); setRecommendation(''); }
  };

  const previewText = `**訪視日期：**${new Date(date).getFullYear()-1911}年${new Date(date).getMonth()+1}月${new Date(date).getDate()}日

**訪視方式：**${method}

**受訪者：**${interviewee}

一、訪視目的：
${purpose}

二、訪視內容：
【健康狀況】
${health}

【服務滿意度評估】
${satisfaction}

【資源連結與後續追蹤】
${special}

三、結論與建議：
${conclusion}`;

  const copyToClipboard = () => {
    navigator.clipboard.writeText(previewText).then(() => {
      alert('已複製紀錄至剪貼簿');
    }).catch(err => {
      alert('複製失敗，請手動複製');
    });
  };

  return (
    <div className="space-y-4 animate-in slide-in-from-bottom-4 duration-300 pb-20">
      <div className="flex justify-between items-center">
        <h2 className="text-xl font-bold text-[#5d4037]">{editRecord ? '編輯紀錄' : '撰寫紀錄'}</h2>
        <div className="flex items-center gap-2">
            {editRecord && <button onClick={onCancel} className="text-xs bg-gray-100 text-gray-500 px-3 py-1 rounded">取消</button>}
            <div className="text-sm font-medium text-[#8d6e63]">{selectedClient ? `(${selectedClient.name})` : '請先選取個案'}</div>
        </div>
      </div>
      <div className="bg-white p-4 rounded-xl shadow-sm border border-[#efebe9] space-y-4">
        <div className="bg-[#fcf9f2] p-3 rounded-lg border border-[#e0e0e0] space-y-3">
          <div className="grid grid-cols-2 gap-3">
            <div>
              <label className="block text-xs font-bold text-[#8d6e63] mb-1">個案</label>
              <select value={clientId} onChange={(e) => setClientId(e.target.value)} disabled={!!editRecord} className="w-full p-2 text-sm border border-[#d7ccc8] rounded bg-white shadow-sm disabled:opacity-50">
                <option value="">-- 選擇個案 --</option>
                {clients.filter(c => c.status === '服務中').map(c => <option key={c.id} value={c.id}>{c.name}</option>)}
              </select>
            </div>
            <div>
              <label className="block text-xs font-bold text-[#8d6e63] mb-1">日期</label>
              <input type="date" value={date} onChange={(e) => setDate(e.target.value)} className="w-full p-2 text-sm border border-[#d7ccc8] rounded bg-white shadow-sm" />
            </div>
          </div>
          <div className="grid grid-cols-2 gap-3">
            <div>
              <label className="block text-xs font-bold text-[#8d6e63] mb-1">方式</label>
              <select value={method} onChange={(e) => setMethod(e.target.value)} className="w-full p-2 text-sm border border-[#d7ccc8] rounded bg-white shadow-sm">
                <option>家庭訪視</option><option>電話訪視</option><option>不定期訪視</option>
              </select>
            </div>
            <div>
              <label className="block text-xs font-bold text-[#8d6e63] mb-1">受訪者</label>
              <select value={interviewee} onChange={(e) => setInterviewee(e.target.value)} className="w-full p-2 text-sm border border-[#d7ccc8] rounded bg-white shadow-sm">
                <option>案主</option><option>家屬 (案妻)</option><option>家屬 (案夫)</option><option>家屬 (案子)</option><option>家屬 (案女)</option><option>其他</option>
              </select>
            </div>
          </div>
        </div>

        <div className="space-y-3">
          <div>
            <label className="block text-xs font-bold text-[#8d6e63] mb-1">一、訪視目的</label>
            <textarea rows={2} placeholder="請輸入簡要目的，AI 將協助轉化為條列式..." value={purpose} onChange={(e) => setPurpose(e.target.value)} className="w-full p-2 text-sm border border-[#d7ccc8] rounded focus:ring-1 focus:ring-[#8d6e63] outline-none" />
          </div>

          <div className="relative">
            <label className="block text-xs font-bold text-[#8d6e63] mb-1">二、觀察資訊 (供 AI 生成參考)</label>
            <textarea rows={2} placeholder="補充個案近期活動、情緒、或生理狀況細節..." value={recentActivities} onChange={(e) => setRecentActivities(e.target.value)} className="w-full p-2 text-sm border border-[#d7ccc8] rounded focus:ring-1 focus:ring-[#8d6e63] outline-none bg-blue-50/30" />
            <button onClick={handleAI} disabled={isGenerating} className={`absolute right-2 bottom-2 text-xs px-4 py-1.5 rounded-full font-bold shadow-md transition-all flex items-center gap-1.5 ${isGenerating ? 'bg-gray-300' : 'bg-gradient-to-r from-[#8d6e63] to-[#a1887f] text-white active:scale-95'}`}>
              {isGenerating ? <div className="animate-spin text-xs">⏳</div> : <><Sparkles size={12} /> AI 生成紀錄</>}
            </button>
          </div>

          <div className="grid grid-cols-1 gap-3">
            <textarea rows={4} placeholder="【健康狀況】(AI 生成)" value={health} onChange={(e) => setHealth(e.target.value)} className="w-full p-2 text-sm border border-[#d7ccc8] rounded outline-none focus:border-[#8d6e63]" />
            <textarea rows={3} placeholder="【服務滿意度】(AI 生成)" value={satisfaction} onChange={(e) => setSatisfaction(e.target.value)} className="w-full p-2 text-sm border border-[#d7ccc8] rounded outline-none focus:border-[#8d6e63]" />
            <textarea rows={3} placeholder="【資源連結與後續追蹤】(AI 生成)" value={special} onChange={(e) => setSpecial(e.target.value)} className="w-full p-2 text-sm border border-[#d7ccc8] rounded outline-none focus:border-[#8d6e63]" />
            <textarea rows={4} placeholder="三、結論與建議 (AI 生成)" value={conclusion} onChange={(e) => setConclusion(e.target.value)} className="w-full p-2 text-sm border border-[#d7ccc8] rounded outline-none focus:border-[#8d6e63]" />
          </div>
        </div>

        <button onClick={handleSave} className="w-full py-3 bg-[#8d6e63] text-white rounded-xl font-bold shadow-md hover:bg-[#5d4037] transition-all flex items-center justify-center gap-2 active:scale-95">
          <Save size={16} /> {editRecord ? '更新紀錄' : '儲存紀錄'}
        </button>
      </div>

      <div className="bg-[#4e342e] text-[#fcf9f2] p-4 rounded-xl shadow-lg mt-4 overflow-hidden relative group">
        <h4 className="text-xs font-bold mb-2 text-[#secondary-color] flex items-center gap-2 opacity-80">
          <FileText size={14} /> 預覽與預計輸出 (自訂格式)
        </h4>
        <div className="bg-white/5 p-3 rounded border border-white/10">
          <pre className="whitespace-pre-wrap font-sans text-[11px] font-medium leading-relaxed max-h-48 overflow-y-auto">
            {previewText}
          </pre>
        </div>
        <button onClick={copyToClipboard} className="absolute top-4 right-4 bg-white/10 hover:bg-white/20 text-white text-[10px] px-2 py-1 rounded border border-white/20 transition opacity-0 group-hover:opacity-100">
            複製全文
        </button>
      </div>
    </div>
  );
};

const Timeline: React.FC<{ clients: Client[]; records: Record<string, VisitRecord[]>; manualTimeline: Record<string, ManualTimelineEntry[]>; onManualToggle: (clientId: string, year: number, month: number, type: 'home' | 'phone' | 'other' | null) => void; onEditRecord: (record: VisitRecord) => void; onDeleteRecord: (clientId: string, recordId: string) => void }> = ({ clients, records, manualTimeline, onManualToggle, onEditRecord, onDeleteRecord }) => {
  const [selectedYear, setSelectedYear] = useState<number>(2025);
  const [selectedCell, setSelectedCell] = useState<{ client: Client, month: { y: number, m: number }, recs: VisitRecord[] } | null>(null);
  const activeClients = clients.filter(c => c.status === ClientStatus.ACTIVE);
  const months = useMemo(() => Array.from({ length: 12 }, (_, i) => ({ y: selectedYear, m: i })), [selectedYear]);

  const handleCellClick = (client: Client, month: { y: number, m: number }) => {
    const clientRecs = records[client.id] || [];
    const filtered = clientRecs.filter(r => { const rd = new Date(r.date); return rd.getFullYear() === month.y && rd.getMonth() === month.m; });
    if (filtered.length > 0) { setSelectedCell({ client, month, recs: filtered }); } 
    else {
      const manualEntries = manualTimeline[client.id] || [];
      const currentManual = manualEntries.find(e => e.year === month.y && e.month === month.m);
      let nextType: 'home' | 'phone' | 'other' | null = null;
      if (!currentManual) nextType = 'home'; else if (currentManual.type === 'home') nextType = 'phone'; else if (currentManual.type === 'phone') nextType = 'other'; else nextType = null;
      onManualToggle(client.id, month.y, month.m, nextType);
    }
  };

  const getCellContent = (client: Client, month: { y: number, m: number }) => {
    const clientRecs = records[client.id] || [];
    const record = clientRecs.find(r => { const rd = new Date(r.date); return rd.getFullYear() === month.y && rd.getMonth() === month.m; });
    if (record) {
      if (record.method.includes('家庭')) return { icon: '🏠', color: 'bg-green-50 text-green-700' };
      if (record.method.includes('電話')) return { icon: '📞', color: 'bg-yellow-50 text-yellow-700' };
      return { icon: '✓', color: 'bg-blue-50 text-blue-700' };
    }
    const manual = (manualTimeline[client.id] || []).find(e => e.year === month.y && e.month === month.m);
    if (manual) {
      if (manual.type === 'home') return { icon: '🏠', color: 'bg-green-50 text-green-400 opacity-60' };
      if (manual.type === 'phone') return { icon: '📞', color: 'bg-yellow-50 text-yellow-400 opacity-60' };
      if (manual.type === 'other') return { icon: '✓', color: 'bg-blue-50 text-blue-400 opacity-60' };
    }
    return { icon: '', color: '' };
  };

  return (
    <div className="space-y-4 animate-in fade-in duration-300 relative">
      <div className="flex justify-between items-center">
        <h2 className="text-xl font-bold text-[#5d4037] flex items-center gap-2"><CalendarCheck className="text-[#8d6e63]" /> 訪視紀錄表</h2>
        <div className="relative">
          <select value={selectedYear} onChange={(e) => setSelectedYear(parseInt(e.target.value))} className="appearance-none bg-white border border-[#d7ccc8] text-[#5d4037] text-xs font-bold py-1.5 pl-3 pr-8 rounded-lg focus:outline-none focus:ring-2 focus:ring-[#8d6e63] shadow-sm">
            {[2025, 2026, 2027].map(y => <option key={y} value={y}>{y}年度</option>)}
          </select>
          <ChevronDown className="absolute right-2 top-1/2 -translate-y-1/2 text-gray-400 pointer-events-none" size={12} />
        </div>
      </div>
      <div className="bg-white rounded-xl shadow-sm border border-[#efebe9] overflow-hidden">
        <div className="overflow-x-auto no-scrollbar">
          <table className="w-full text-left border-collapse min-w-[700px]">
            <thead>
              <tr className="bg-[#d7ccc8]">
                <th className="sticky left-0 bg-[#d7ccc8] z-20 text-[10px] p-2 border-r border-[#efebe9] min-w-[80px] text-[#5d4037]">個案姓名</th>
                {months.map((month, idx) => <th key={idx} className="text-[10px] p-2 text-center border-r border-[#efebe9] min-w-[50px] whitespace-nowrap text-[#5d4037]">{month.m + 1}月</th>)}
              </tr>
            </thead>
            <tbody>
              {activeClients.map(client => (
                <tr key={client.id} className="border-b border-gray-50 hover:bg-gray-50 transition-colors">
                  <td className="sticky left-0 bg-white z-10 p-2 text-[12px] font-bold border-r border-[#efebe9] truncate max-w-[80px] text-[#4e342e]">{client.name}</td>
                  {months.map((month, idx) => {
                    const { icon, color } = getCellContent(client, month);
                    return <td key={idx} onClick={() => handleCellClick(client, month)} className={`text-center text-[16px] border-r border-[#efebe9] p-2 cursor-pointer transition-colors h-11 ${color}`}>{icon}</td>;
                  })}
                </tr>
              ))}
            </tbody>
          </table>
        </div>
      </div>
      {selectedCell && (
        <div className="fixed inset-0 bg-black/60 backdrop-blur-sm z-[100] flex items-center justify-center p-6 animate-in fade-in duration-200">
          <div className="bg-white w-full max-w-sm rounded-2xl shadow-2xl flex flex-col max-h-[80vh] overflow-hidden">
            <div className="p-4 bg-[#fcf9f2] border-b border-[#efebe9] flex justify-between items-center">
              <div><h3 className="font-bold text-[#5d4037]">{selectedCell.client.name}</h3><p className="text-[10px] text-gray-400">{selectedCell.month.y}年{selectedCell.month.m + 1}月 訪視詳情</p></div>
              <button onClick={() => setSelectedCell(null)} className="text-gray-400 p-1 hover:bg-gray-100 rounded-full"><X size={20} /></button>
            </div>
            <div className="p-4 overflow-y-auto space-y-4 no-scrollbar bg-white">
              {selectedCell.recs.map(rec => (
                <div key={rec.id} className="p-3 border border-[#efebe9] rounded-xl bg-gray-50 space-y-2 relative group shadow-sm">
                  <div className="flex justify-between items-start"><span className="text-[10px] font-bold px-2 py-0.5 bg-[#8d6e63] text-white rounded-full">{rec.method}</span><span className="text-[10px] text-gray-400 font-medium">{rec.displayDate}</span></div>
                  <div className="text-xs text-[#4e342e] border-l-2 border-[#d7ccc8] pl-2 py-1 bg-white/50 rounded"><p className="font-bold mb-1 text-[10px] text-[#8d6e63] uppercase tracking-wider">訪視目的</p><p className="line-clamp-3 leading-relaxed">{rec.purpose}</p></div>
                  <div className="flex gap-2 pt-2">
                    <button onClick={() => onEditRecord(rec)} className="flex-1 py-2 bg-white border border-[#d7ccc8] text-[#8d6e63] rounded-lg text-[10px] font-bold active:scale-95 transition shadow-sm hover:border-[#8d6e63] flex items-center justify-center gap-1"><Pencil size={12} /> 編輯紀錄</button>
                    <button onClick={() => { onDeleteRecord(rec.clientId, rec.id); setSelectedCell(null); }} className="px-3 py-2 bg-red-50 text-red-400 border border-red-100 rounded-lg text-[10px] active:scale-95 transition hover:bg-red-100"><Trash2 size={12} /></button>
                  </div>
                </div>
              ))}
            </div>
          </div>
        </div>
      )}
    </div>
  );
};

const ServiceCalculator: React.FC = () => {
  const [welfare, setWelfare] = useState<WelfareType>('normal');
  const [selectedItems, setSelectedItems] = useState<Record<string, number>>({});
  const [showResult, setShowResult] = useState(false);

  const sortedItems = useMemo(() => [...SERVICE_ITEMS].sort((a, b) => a.code.localeCompare(b.code)), []);
  const updateQuantity = (code: string, delta: number) => {
    setShowResult(false);
    setSelectedItems(prev => { const current = prev[code] || 0; const next = Math.max(0, current + delta); if (next === 0) { const { [code]: _, ...rest } = prev; return rest; } return { ...prev, [code]: next }; });
  };
  const summary = useMemo(() => {
    let totalPay = 0, totalSelf = 0;
    const items = Object.entries(selectedItems).map(([code, qty]) => {
      const item = SERVICE_ITEMS.find(i => i.code === code)!;
      const pay = item.pay * qty; const self = item.cost[welfare] * qty;
      totalPay += pay; totalSelf += self;
      return { ...item, qty, payTotal: pay, selfTotal: self };
    });
    return { items, totalPay, totalSelf };
  }, [selectedItems, welfare]);

  return (
    <div className="space-y-4 animate-in fade-in duration-300 pb-24">
      <h2 className="text-xl font-bold text-[#5d4037] flex items-center gap-2"><Calculator className="text-[#8d6e63]" /> 服務項目試算</h2>
      <div className="bg-white p-3 rounded-xl shadow-sm border border-[#efebe9] flex gap-2">
        {(['normal', 'midLow', 'low'] as WelfareType[]).map(type => (
          <button key={type} onClick={() => { setWelfare(type); setShowResult(false); }} className={`flex-1 py-2 text-xs font-bold rounded-lg transition-all border ${welfare === type ? 'bg-[#8d6e63] text-white border-[#8d6e63]' : 'bg-gray-50 text-gray-400 border-gray-200'}`}>
            {type === 'normal' ? '一般戶' : type === 'midLow' ? '中低收' : '低收入'}
          </button>
        ))}
      </div>
      <div className="space-y-2 max-h-[50vh] overflow-y-auto no-scrollbar pr-1">
        {sortedItems.map(item => (
          <div key={item.code} className="bg-white p-3 rounded-xl border border-[#efebe9] shadow-sm flex items-center justify-between transition-all hover:border-[#8d6e63]/30">
            <div className="flex-1">
              <div className="flex items-center gap-2">
                <span className="text-[10px] font-bold px-1.5 py-0.5 bg-[#fcf9f2] border border-[#d7ccc8] text-[#8d6e63] rounded">{item.code}</span>
                <span className="text-sm font-bold text-[#4e342e]">{item.name}</span>
              </div>
              <div className="text-[10px] text-gray-400 mt-1">單次給付 ${item.pay.toLocaleString()} | 自付 ${item.cost[welfare].toLocaleString()}</div>
            </div>
            <div className="flex items-center gap-3">
              <button onClick={() => updateQuantity(item.code, -1)} className="w-8 h-8 flex items-center justify-center rounded-full bg-gray-50 text-[#8d6e63] border border-gray-100 active:scale-90 transition disabled:opacity-20" disabled={!selectedItems[item.code]}><Minus size={14} /></button>
              <span className="text-sm font-bold w-4 text-center">{selectedItems[item.code] || 0}</span>
              <button onClick={() => updateQuantity(item.code, 1)} className="w-8 h-8 flex items-center justify-center rounded-full bg-[#8d6e63] text-white shadow-sm active:scale-90 transition"><Plus size={14} /></button>
            </div>
          </div>
        ))}
      </div>
      <div className="pt-2">
        <button onClick={() => { if (Object.keys(selectedItems).length > 0) setShowResult(true); else alert('請選擇項目'); }} className="w-full py-4 bg-[#8d6e63] text-white rounded-2xl font-bold shadow-lg shadow-[#8d6e63]/20 active:scale-95 transition flex items-center justify-center gap-2">
          <CheckCircle size={20} /> 產出試算結果
        </button>
      </div>
      {showResult && Object.keys(selectedItems).length > 0 && (
        <div className="fixed inset-x-4 bottom-24 max-w-[calc(448px-2rem)] mx-auto z-40 animate-in slide-in-from-bottom-10 duration-300">
          <div className="bg-[#4e342e] text-white p-5 rounded-2xl shadow-2xl space-y-4 border border-white/10 backdrop-blur-md">
            <div className="flex justify-between items-center border-b border-white/10 pb-3">
              <div><p className="text-[10px] text-white/60 mb-1">SELECTED {Object.keys(selectedItems).length} ITEMS</p><p className="text-lg font-bold">試算結果總結</p></div>
              <button onClick={() => { setSelectedItems({}); setShowResult(false); }} className="text-[10px] bg-white/10 px-3 py-1.5 rounded-full hover:bg-white/20 transition border border-white/20">清除內容</button>
            </div>
            <div className="grid grid-cols-2 gap-6">
              <div className="space-y-1"><p className="text-[10px] text-white/50 uppercase tracking-widest">總給付額度</p><p className="text-2xl font-bold text-[#d7ccc8]">${summary.totalPay.toLocaleString()}</p></div>
              <div className="text-right space-y-1"><p className="text-[10px] text-white/50 uppercase tracking-widest">預估總自付</p><p className="text-2xl font-bold text-yellow-400">${summary.totalSelf.toLocaleString()}</p></div>
            </div>
            <div className="max-h-32 overflow-y-auto no-scrollbar pt-2 space-y-2 border-t border-white/5">
              {summary.items.map(i => <div key={i.code} className="flex justify-between text-[11px] opacity-90 border-l-2 border-[#8d6e63] pl-2 py-0.5"><span className="font-medium">{i.code} {i.name} <span className="text-white/40">x {i.qty}</span></span><span className="font-mono text-yellow-100/70">${i.selfTotal.toLocaleString()}</span></div>)}
            </div>
          </div>
        </div>
      )}
    </div>
  );
};

const SOPLookup: React.FC<{ sopData: Record<string, SOPItem[]>; onUpdate: (item: SOPItem) => void; onDelete: (id: string, category: string) => void }> = ({ sopData, onUpdate, onDelete }) => {
  const [activeTab, setActiveTab] = useState<'routine' | 'govt' | 'compal'>('routine');
  const [editingItem, setEditingItem] = useState<SOPItem | null>(null);

  const currentItems = sopData[activeTab] || [];

  return (
    <div className="space-y-4 animate-in fade-in duration-300 pb-20">
      <div className="flex justify-between items-center">
        <h2 className="text-xl font-bold text-[#5d4037] flex items-center gap-2"><BookOpen className="text-[#8d6e63]" /> 操作 SOP 速查</h2>
        <button onClick={() => setEditingItem({ id: Date.now().toString(), category: activeTab, title: '', desc: '' })} className="text-xs bg-[#8d6e63] text-white px-3 py-1.5 rounded-full font-bold shadow-sm active:scale-95 transition flex items-center gap-1"><Plus size={12} /> 新增</button>
      </div>
      <div className="flex space-x-2 bg-white p-1 rounded-xl border border-[#efebe9] sticky top-0 z-10 shadow-sm">
        {[{ id: 'routine', label: '例行工作' }, { id: 'govt', label: '照管系統' }, { id: 'compal', label: '仁寶系統' }].map(tab => (
          <button key={tab.id} onClick={() => setActiveTab(tab.id as any)} className={`flex-1 py-2 text-xs font-bold rounded-lg transition-all ${activeTab === tab.id ? 'bg-[#8d6e63] text-white shadow' : 'text-gray-400 hover:bg-gray-50'}`}>{tab.label}</button>
        ))}
      </div>
      <div className="space-y-4">
        {currentItems.length === 0 ? <div className="text-center py-12 text-gray-400 text-sm italic">此分類尚無 SOP 內容</div> : currentItems.map((item) => (
          <div key={item.id} className="group bg-white p-4 rounded-xl shadow-sm border-l-4 border-[#8d6e63] hover:shadow-md transition relative">
            <div className="absolute right-2 top-2 opacity-0 group-hover:opacity-100 transition-opacity flex gap-2">
              <button onClick={() => setEditingItem({ ...item })} className="p-1.5 text-[#8d6e63] hover:bg-gray-50 rounded"><Pencil size={14} /></button>
              <button onClick={() => confirm('確定刪除？') && onDelete(item.id, activeTab)} className="p-1.5 text-red-400 hover:bg-red-50 rounded"><Trash2 size={14} /></button>
            </div>
            <h3 className="font-bold text-[#5d4037] mb-2 flex items-center gap-2 text-sm pr-12"><ChevronRight size={14} className="text-[#8d6e63]" /> {item.title}</h3>
            <p className="text-xs text-gray-600 leading-relaxed whitespace-pre-wrap">{item.desc}</p>
          </div>
        ))}
      </div>
      {editingItem && (
        <div className="fixed inset-0 bg-black/60 backdrop-blur-sm z-[110] flex items-center justify-center p-6">
          <div className="bg-white w-full max-w-sm rounded-2xl shadow-2xl animate-in zoom-in-95 duration-200">
            <div className="p-4 bg-[#fcf9f2] border-b border-[#efebe9] flex justify-between items-center rounded-t-2xl"><h3 className="font-bold text-[#5d4037]">編輯 SOP</h3><button onClick={() => setEditingItem(null)} className="text-gray-400"><X size={20} /></button></div>
            <div className="p-6 space-y-4">
              <div><label className="block text-xs font-bold text-gray-500 mb-1">標題</label><input value={editingItem.title} onChange={e => setEditingItem({...editingItem, title: e.target.value})} className="w-full p-2 border rounded border-[#d7ccc8] text-sm" /></div>
              <div><label className="block text-xs font-bold text-gray-500 mb-1">分類</label><select value={editingItem.category} onChange={e => setEditingItem({...editingItem, category: e.target.value as any})} className="w-full p-2 border rounded border-[#d7ccc8] text-sm"><option value="routine">例行工作</option><option value="govt">照管系統</option><option value="compal">仁寶系統</option></select></div>
              <div><label className="block text-xs font-bold text-gray-500 mb-1">內容</label><textarea value={editingItem.desc} onChange={e => setEditingItem({...editingItem, desc: e.target.value})} rows={6} className="w-full p-2 border rounded border-[#d7ccc8] text-sm" /></div>
              <div className="flex gap-3 pt-2"><button onClick={() => setEditingItem(null)} className="flex-1 py-2 bg-gray-100 text-gray-500 rounded-xl font-bold">取消</button><button onClick={() => { onUpdate(editingItem); setEditingItem(null); }} className="flex-1 py-2 bg-[#8d6e63] text-white rounded-xl font-bold">儲存</button></div>
            </div>
          </div>
        </div>
      )}
    </div>
  );
};

const ImportModal: React.FC<{ onClose: () => void; onImport: (clients: Client[]) => void; onManualAdd: () => void }> = ({ onClose, onImport, onManualAdd }) => {
  const [status, setStatus] = useState('');
  const [xlsxData, setXlsxData] = useState<any[]>([]);

  const handleXlsxChange = async (e: React.ChangeEvent<HTMLInputElement>) => {
    const file = e.target.files?.[0]; if (!file) return;
    setStatus('正在讀取檔案與偵測編碼...');
    try {
      const data = await parseFile(file);
      setXlsxData(data);
      if (data.length === 0) {
        setStatus('讀取失敗：無法解析資料，請確認檔案格式是否正確 (CSV/Excel)');
      } else {
        setStatus(`讀取成功：${data.length} 筆資料`);
      }
    } catch (err) { 
      setStatus('讀取失敗，請確認檔案格式'); 
    }
  };

  const confirmXlsx = () => {
    onImport(xlsxData); onClose();
  };

  const handleHtmlChange = async (e: React.ChangeEvent<HTMLInputElement>) => {
    const file = e.target.files?.[0]; if (!file) return;
    try { const text = await file.text(); const client = parseHtmlReport(text); onImport([client as Client]); onClose(); } catch (err) { setStatus('HTML 解析失敗'); }
  };

  return (
    <div className="fixed inset-0 bg-black/60 backdrop-blur-sm z-[100] flex items-center justify-center p-6 animate-in fade-in duration-200">
      <div className="bg-white w-full max-w-sm rounded-2xl shadow-2xl flex flex-col max-h-[90vh]">
        <div className="p-4 bg-[#fcf9f2] border-b border-[#efebe9] flex justify-between items-center rounded-t-2xl"><h3 className="font-bold text-[#5d4037]">資料匯入中心</h3><button onClick={onClose} className="text-gray-400 hover:text-gray-600"><X size={20} /></button></div>
        <div className="p-6 space-y-4 overflow-y-auto no-scrollbar">
          <div className="bg-[#efebe9]/50 p-3 rounded-xl border border-[#d7ccc8]">
            <h4 className="text-xs font-bold text-[#5d4037] mb-2 flex items-center gap-1"><FileSpreadsheet size={14} className="text-[#8d6e63]" /> 仁寶 Excel/CSV</h4>
            <input type="file" accept=".xlsx,.csv" onChange={handleXlsxChange} className="hidden" id="xlsx-input" />
            <label htmlFor="xlsx-input" className="block text-center py-2 bg-white border border-[#d7ccc8] rounded-lg text-[10px] cursor-pointer hover:bg-gray-50">選擇檔案 (Excel或CSV)</label>
            {xlsxData.length > 0 && (<div className="mt-2 space-y-2"><button onClick={confirmXlsx} className="w-full py-2 bg-[#8d6e63] text-white text-[10px] font-bold rounded">確認匯入 ({xlsxData.length}筆)</button></div>)}
          </div>
          <div className="bg-white p-3 rounded-xl border border-[#efebe9]">
            <h4 className="text-xs font-bold text-[#5d4037] mb-2 flex items-center gap-1"><FileCode size={14} className="text-blue-500" /> 衛福部 HTML 報表</h4>
            <input type="file" accept=".html,.htm" onChange={handleHtmlChange} className="hidden" id="html-input" />
            <label htmlFor="html-input" className="block text-center py-2 border border-[#d7ccc8] rounded-lg text-[10px] cursor-pointer">上傳 HTML</label>
          </div>
          <p className="text-center text-[10px] text-[#8d6e63] font-medium">{status}</p>
          <button onClick={onManualAdd} className="w-full py-2 bg-[#d7ccc8] text-[#5d4037] rounded-xl font-bold text-xs active:scale-95 transition">手動新增個案</button>
        </div>
      </div>
    </div>
  );
};

const EditClientModal: React.FC<{ client: Client; onClose: () => void; onSave: (client: Client) => void }> = ({ client, onClose, onSave }) => {
  const [formData, setFormData] = useState<Client>({ ...client });
  const handleChange = (e: React.ChangeEvent<HTMLInputElement | HTMLSelectElement>) => { const { name, value } = e.target; setFormData(prev => ({ ...prev, [name]: value })); };
  return (
    <div className="fixed inset-0 bg-black/60 backdrop-blur-sm z-[110] flex items-center justify-center p-6 animate-in fade-in zoom-in-95 duration-200">
      <div className="bg-white w-full max-w-sm rounded-2xl shadow-2xl flex flex-col max-h-[90vh]">
        <div className="p-4 bg-[#fcf9f2] border-b border-[#efebe9] flex justify-between items-center rounded-t-2xl"><h3 className="font-bold text-[#5d4037]">編輯個案資料</h3><button onClick={onClose} className="text-gray-400 hover:text-gray-600"><X size={20} /></button></div>
        <div className="p-6 space-y-4 overflow-y-auto no-scrollbar text-sm">
          <div><label className="block text-xs font-bold text-gray-500 mb-1">姓名</label><input name="name" value={formData.name} onChange={handleChange} className="w-full p-2 border rounded border-[#d7ccc8]" /></div>
          <div className="grid grid-cols-2 gap-2">
            <div><label className="block text-xs font-bold text-gray-500 mb-1">性別</label><select name="gender" value={formData.gender} onChange={handleChange} className="w-full p-2 border rounded border-[#d7ccc8]"><option value="男性">男性</option><option value="女性">女性</option><option value="未知">未知</option></select></div>
            <div><label className="block text-xs font-bold text-gray-500 mb-1">狀態</label><select name="status" value={formData.status} onChange={handleChange} className="w-full p-2 border rounded border-[#d7ccc8]">{Object.values(ClientStatus).map(s => <option key={s} value={s}>{s}</option>)}</select></div>
          </div>
          <div className="grid grid-cols-2 gap-2">
            <div><label className="block text-xs font-bold text-gray-500 mb-1">年齡</label><input name="age" value={formData.age || ''} onChange={handleChange} className="w-full p-2 border rounded border-[#d7ccc8]" /></div>
            <div><label className="block text-xs font-bold text-gray-500 mb-1">電話</label><input name="phone" value={formData.phone} onChange={handleChange} className="w-full p-2 border rounded border-[#d7ccc8]" /></div>
          </div>
          <div><label className="block text-xs font-bold text-gray-500 mb-1">地址</label><input name="address" value={formData.address} onChange={handleChange} className="w-full p-2 border rounded border-[#d7ccc8]" /></div>
          <div className="grid grid-cols-2 gap-2">
            <div><label className="block text-xs font-bold text-gray-500 mb-1">福利身分</label><input name="welfare" value={formData.welfare} onChange={handleChange} className="w-full p-2 border rounded border-[#d7ccc8]" /></div>
            <div><label className="block text-xs font-bold text-gray-500 mb-1">主責督導</label><input name="supervisor" value={formData.supervisor || ''} onChange={handleChange} className="w-full p-2 border rounded border-[#d7ccc8]" /></div>
          </div>
          <div className="pt-4 flex gap-3"><button onClick={onClose} className="flex-1 py-2 bg-gray-100 text-gray-500 rounded-xl font-bold">取消</button><button onClick={() => { if(!formData.name) return; onSave(formData); }} className="flex-1 py-2 bg-[#8d6e63] text-white rounded-xl font-bold">儲存</button></div>
        </div>
      </div>
    </div>
  );
};

const App: React.FC = () => {
  const [activeTab, setActiveTab] = useState<ViewType>('home');
  const [clients, setClients] = useState<Client[]>([]);
  const [records, setRecords] = useState<Record<string, VisitRecord[]>>({});
  const [manualTimeline, setManualTimeline] = useState<Record<string, ManualTimelineEntry[]>>({});
  const [sopData, setSopData] = useState<Record<string, SOPItem[]>>({});
  const [config, setConfig] = useState<AppConfig>({ apiKey: '', gasUrl: '', syncId: '' });
  const [user, setUser] = useState<User | null>(null);
  
  const [isSettingsOpen, setIsSettingsOpen] = useState(false);
  const [isImportOpen, setIsImportOpen] = useState(false);
  const [editingClient, setEditingClient] = useState<Client | null>(null);
  const [editingRecord, setEditingRecord] = useState<VisitRecord | null>(null);
  const [selectedClientId, setSelectedClientId] = useState<string | null>(null);

  useEffect(() => {
    if (!(window as any).XLSX) {
      const script = document.createElement('script');
      script.src = "https://cdn.sheetjs.com/xlsx-0.20.1/package/dist/xlsx.full.min.js";
      script.async = true;
      document.body.appendChild(script);
    }
  }, []);

  useEffect(() => {
    if (!auth) return;
    const initAuth = async () => {
      // @ts-ignore
      if (typeof __initial_auth_token !== 'undefined' && __initial_auth_token) {
        // @ts-ignore
        try {
            await signInWithCustomToken(auth, __initial_auth_token);
        } catch (e) {
            console.warn("Custom token auth failed (likely config mismatch), falling back to anonymous", e);
            await signInAnonymously(auth);
        }
      } else {
        await signInAnonymously(auth);
      }
    };
    initAuth();
    return onAuthStateChanged(auth, setUser);
  }, []);

  // --- Path Helper: SyncId vs Private ---
  const getCollectionRef = (colName: string) => {
    if (config.syncId) {
       // Use PUBLIC path with a prefixed collection name to simulate grouping
       return collection(db, 'artifacts', appId, 'public', 'data', `${config.syncId}_${colName}`);
    } else {
       // Use PRIVATE user path
       return collection(db, 'artifacts', appId, 'users', user?.uid || 'unknown', colName);
    }
  };

  useEffect(() => {
    if (!user || !db) return;

    // We must listen to config first to know if we have a syncId
    // But config is always stored in private user space for security settings
    const unsubConfig = onSnapshot(doc(db, 'artifacts', appId, 'users', user.uid, 'config', 'main'), (snap) => {
      if (snap.exists()) setConfig(snap.data() as AppConfig);
    });

    return () => unsubConfig();
  }, [user]);

  // Main Data Sync
  useEffect(() => {
    if (!user || !db) return;

    // Switch data source based on config.syncId
    const clientsRef = getCollectionRef('clients');
    const recordsRef = getCollectionRef('records');
    const timelineRef = getCollectionRef('timeline');
    const sopRef = getCollectionRef('sop');

    const unsubClients = onSnapshot(clientsRef, (snap) => {
      const list: Client[] = [];
      snap.forEach(d => list.push(d.data() as Client));
      setClients(list);
    });

    const unsubRecords = onSnapshot(recordsRef, (snap) => {
      const map: Record<string, VisitRecord[]> = {};
      snap.forEach(d => {
        const r = d.data() as VisitRecord;
        if (!map[r.clientId]) map[r.clientId] = [];
        map[r.clientId].push(r);
      });
      Object.keys(map).forEach(k => {
        map[k].sort((a, b) => new Date(b.date).getTime() - new Date(a.date).getTime());
      });
      setRecords(map);
    });

    const unsubTimeline = onSnapshot(timelineRef, (snap) => {
      const map: Record<string, ManualTimelineEntry[]> = {};
      snap.forEach(d => {
        const t = d.data() as ManualTimelineEntry;
        if (!map[t.clientId]) map[t.clientId] = [];
        map[t.clientId].push(t);
      });
      setManualTimeline(map);
    });

    const unsubSop = onSnapshot(sopRef, (snap) => {
      const map: Record<string, SOPItem[]> = {};
      ['routine', 'govt', 'compal'].forEach(c => map[c] = []);
      snap.forEach(d => {
        const s = d.data() as SOPItem;
        if (!map[s.category]) map[s.category] = [];
        map[s.category].push(s);
      });
      setSopData(map);
    });

    return () => {
      unsubClients();
      unsubRecords();
      unsubTimeline();
      unsubSop();
    };
  }, [user, config.syncId]); // Re-run when syncId changes

  // --- Handlers ---

  const handleSaveClient = async (client: Client) => {
    if (!user || !db) return;
    const ref = doc(getCollectionRef('clients'), client.id);
    await setDoc(ref, client);
    setEditingClient(null);
  };

  const handleDeleteClient = async (id: string) => {
    if (!user || !db) return;
    const ref = doc(getCollectionRef('clients'), id);
    await deleteDoc(ref);
  };

  const handleImportClients = async (newClients: Client[]) => {
    if (!user || !db) return;
    const batch = writeBatch(db);
    newClients.forEach(c => {
      const ref = doc(getCollectionRef('clients'), c.id);
      batch.set(ref, c);
    });
    await batch.commit();
    setIsImportOpen(false);
  };

  const handleSaveRecord = async (record: VisitRecord) => {
    if (!user || !db) return;
    const ref = doc(getCollectionRef('records'), record.id);
    await setDoc(ref, record);
    if (record.method === '家庭訪視') {
      const client = clients.find(c => c.id === record.clientId);
      if (client) {
        const cRef = doc(getCollectionRef('clients'), client.id);
        await setDoc(cRef, { ...client, last_visit_date: record.date });
      }
    }
    setEditingRecord(null);
  };

  const handleDeleteRecord = async (clientId: string, recordId: string) => {
    if (!user || !db) return;
    const ref = doc(getCollectionRef('records'), recordId);
    await deleteDoc(ref);
  };

  const handleTimelineToggle = async (clientId: string, year: number, month: number, type: 'home' | 'phone' | 'other' | null) => {
    if (!user || !db) return;
    const id = `${clientId}_${year}_${month}`;
    const ref = doc(getCollectionRef('timeline'), id);
    if (type === null) {
      await deleteDoc(ref);
    } else {
      await setDoc(ref, { clientId, year, month, type });
    }
  };

  const handleSOPUpdate = async (item: SOPItem) => {
    if (!user || !db) return;
    const ref = doc(getCollectionRef('sop'), item.id);
    await setDoc(ref, item);
  };

  const handleSOPDelete = async (id: string, category: string) => {
    if (!user || !db) return;
    const ref = doc(getCollectionRef('sop'), id);
    await deleteDoc(ref);
  };

  const handleConfigSave = async (newConfig: AppConfig) => {
    if (!user || !db) return;
    // Config always stays private
    await setDoc(doc(db, 'artifacts', appId, 'users', user.uid, 'config', 'main'), newConfig);
    setIsSettingsOpen(false);
  };

  const handleUploadLocal = async () => {
    if (!user || !db) return;
    if (!confirm('確定要將本機資料上傳覆蓋雲端嗎？')) return;
    
    const localClients = JSON.parse(localStorage.getItem('care_helper_clients') || '[]');
    const localRecsMap = JSON.parse(localStorage.getItem('care_helper_records') || '{}');
    const localSopMap = JSON.parse(localStorage.getItem('care_helper_sop') || '{}');
    const localApiKey = localStorage.getItem('care_helper_api_key') || '';
    const localGasUrl = localStorage.getItem('care_helper_gas_url') || '';

    // Upload to whatever is the CURRENT active path (Private or SyncId)
    const batch = writeBatch(db); 
    // Batch limit is 500, simple implementation:
    // We'll just do one-by-one for robustness in this simple demo, or small batches if needed.
    // Given the constraints, let's do direct writes to avoid batch complexity limits for now.
    
    for (const c of localClients) await setDoc(doc(getCollectionRef('clients'), c.id), c);
    
    for (const cid in localRecsMap) {
      for (const r of localRecsMap[cid]) await setDoc(doc(getCollectionRef('records'), r.id), r);
    }

    for (const cat in localSopMap) {
      for (const s of localSopMap[cat]) await setDoc(doc(getCollectionRef('sop'), s.id), s);
    }

    // Also update config with local keys if they exist and remote is empty
    if ((localApiKey || localGasUrl) && !config.apiKey) {
      await setDoc(doc(db, 'artifacts', appId, 'users', user.uid, 'config', 'main'), { ...config, apiKey: localApiKey, gasUrl: localGasUrl });
    }

    alert('本機資料已上傳至目前選定的雲端空間');
  };

  const stats = useMemo(() => {
    const active = clients.filter(c => c.status === ClientStatus.ACTIVE).length;
    const now = new Date();
    let monthlyVisits = 0;
    Object.values(records).forEach(clientRecs => {
      clientRecs.forEach(rec => { if (new Date(rec.date).getMonth() === now.getMonth()) monthlyVisits++; });
    });
    return { active, monthlyVisits };
  }, [clients, records]);

  return (
    <div className="flex flex-col h-screen overflow-hidden max-w-md mx-auto bg-[#fcf9f2] shadow-2xl relative font-sans text-[#4e342e]">
      <header className="bg-[#8d6e63] text-white px-4 py-3 flex items-center justify-between shadow-md pt-safe z-30 shrink-0">
        <h1 className="font-bold text-lg flex items-center gap-2"><BriefcaseMedical /> 督導助理</h1>
        <div className="flex items-center gap-3">
          <div className={`p-1 rounded-full ${user ? 'bg-green-400/20 text-green-100' : 'bg-red-400/20 text-red-100'}`}>
            {user ? <Wifi size={14} /> : <WifiOff size={14} />}
          </div>
          <button onClick={() => setIsSettingsOpen(true)} className="bg-white/20 p-2 rounded-full hover:bg-white/30 transition"><Settings size={20} /></button>
          <span className="text-[10px] bg-white/20 px-2 py-0.5 rounded">v22.0</span>
        </div>
      </header>
      <main className="flex-1 overflow-y-auto p-4 no-scrollbar pb-24">
        {activeTab === 'home' && <Dashboard stats={stats} gasUrl={config.gasUrl} clients={clients} records={records} syncId={config.syncId} />}
        {activeTab === 'clients' && <ClientList clients={clients} onDeleteClient={handleDeleteClient} onWriteRecord={(id) => { setSelectedClientId(id); setEditingRecord(null); setActiveTab('record'); }} onImport={() => setIsImportOpen(true)} onEditClient={setEditingClient} />}
        {activeTab === 'timeline' && <Timeline clients={clients} records={records} manualTimeline={manualTimeline} onManualToggle={handleTimelineToggle} onEditRecord={(rec) => { setEditingRecord(rec); setSelectedClientId(rec.clientId); setActiveTab('record'); }} onDeleteRecord={handleDeleteRecord} />}
        {activeTab === 'record' && <RecordForm clients={clients} initialClientId={selectedClientId} editRecord={editingRecord} apiKey={config.apiKey} onSave={handleSaveRecord} onCancel={() => setActiveTab('timeline')} />}
        {activeTab === 'calculator' && <ServiceCalculator />}
        {activeTab === 'sop' && <SOPLookup sopData={sopData} onUpdate={handleSOPUpdate} onDelete={handleSOPDelete} />}
      </main>
      <nav className="fixed bottom-0 left-0 right-0 max-w-md mx-auto bg-white border-t border-gray-100 flex justify-between p-1 pb-safe z-50 shadow-lg shrink-0">
        {[
          { id: 'home', icon: LayoutDashboard, label: '首頁' },
          { id: 'clients', icon: Users, label: '個案' },
          { id: 'timeline', icon: CalendarCheck, label: '訪視表' },
          { id: 'calculator', icon: Calculator, label: '試算' },
          { id: 'record', icon: FileText, label: '紀錄' },
          { id: 'sop', icon: BookOpen, label: 'SOP' }
        ].map(tab => {
          const Icon = tab.icon;
          return (
            <button key={tab.id} onClick={() => { setActiveTab(tab.id as ViewType); if (tab.id !== 'record') setEditingRecord(null); }} className={`flex flex-col items-center p-2 rounded-lg flex-1 transition ${activeTab === tab.id ? 'text-[#8d6e63]' : 'text-gray-400'}`}>
              <Icon size={24} className={activeTab === tab.id ? 'fill-current' : ''} />
              <span className="text-[9px] font-bold mt-0.5">{tab.label}</span>
            </button>
          );
        })}
      </nav>
      {isSettingsOpen && <SettingsModal config={config} onClose={() => setIsSettingsOpen(false)} onSave={handleConfigSave} onUploadLocal={handleUploadLocal} />}
      {isImportOpen && <ImportModal onClose={() => setIsImportOpen(false)} onImport={handleImportClients} onManualAdd={() => { setIsImportOpen(false); setTimeout(() => setEditingClient({ id: `m-${Date.now()}`, name: '', gender: '未知', status: ClientStatus.ACTIVE, welfare: '一般戶', phone: '', address: '', freq_type: '電電家' }), 100); }} />}
      {editingClient && <EditClientModal client={editingClient} onClose={() => setEditingClient(null)} onSave={handleSaveClient} />}
    </div>
  );
};

export default App;
