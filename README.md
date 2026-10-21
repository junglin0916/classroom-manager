
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0" />
  <title>Multi-Class Learning Manager</title>
  <!-- Tailwind CSS CDN -->
  <script src="https://cdn.tailwindcss.com"></script>
  <!-- React & ReactDOM CDN -->
  <script src="https://unpkg.com/react@18/umd/react.development.js" crossorigin></script>
  <script src="https://unpkg.com/react-dom@18/umd/react-dom.development.js" crossorigin></script>
  <!-- Babel CDN -->
  <script src="https://unpkg.com/@babel/standalone/babel.min.js"></script>
</head>
<body class="bg-slate-50">
  <div id="root"></div>

  <script type="text/babel">
    const { useState, useEffect, useRef } = React;

    // Default Initial Data Structure for 5 Classes
    const INITIAL_CLASSES_DATA = {
      'c101': {
        name: 'Class 701',
        groups: [
          { id: 'g1', name: 'Group Alpha', members: [{ id: 's1', name: 'Alice', points: 0 }, { id: 's2', name: 'Bob', points: 0 }] },
          { id: 'g2', name: 'Group Beta', members: [{ id: 's3', name: 'Charlie', points: 0 }] }
        ],
        logs: []
      },
      'c102': { name: 'Class 702', groups: [], logs: [] },
      'c103': { name: 'Class 703', groups: [], logs: [] },
      'c104': { name: 'Class 704', groups: [], logs: [] },
      'c105': { name: 'Class 705', groups: [], logs: [] }
    };

    const INITIAL_CATEGORIES = [
      { id: 'cat1', name: 'Classroom Participation', type: 'good', defaultPoints: 2 },
      { id: 'cat2', name: 'Helping Team Members', type: 'good', defaultPoints: 3 },
      { id: 'cat3', name: 'Class Disruption / Interrupting', type: 'bad', defaultPoints: -2 },
      { id: 'cat4', name: 'Late / Unprepared', type: 'bad', defaultPoints: -1 }
    ];

    function GroupClassroomManager() {
      // --- SECURITY & TIME CONFIGURATION ---
      const MASTER_PASSWORD = "mysecretpassword"; // Change your password here
      const START_HOUR = 8;  // 08:00 AM
      const END_HOUR = 17;   // 05:00 PM (17:00)

      // State Management with LocalStorage persistence
      const [classesData, setClassesData] = useState(() => {
        const saved = localStorage.getItem('GCM_CLASSES_DATA');
        return saved ? JSON.parse(saved) : INITIAL_CLASSES_DATA;
      });

      const [selectedClassId, setSelectedClassId] = useState('c101');

      const [categories, setCategories] = useState(() => {
        const saved = localStorage.getItem('GCM_CATEGORIES');
        return saved ? JSON.parse(saved) : INITIAL_CATEGORIES;
      });

      // Security States
      const [inputPassword, setInputPassword] = useState('');
      const [isAuthenticated, setIsAuthenticated] = useState(false);
      const [isTimeAllowed, setIsTimeAllowed] = useState(false);
      const [timeStatusMessage, setTimeStatusMessage] = useState('');

      // Form Inputs
      const [newGroupName, setNewGroupName] = useState('');
      const [selectedGroupId, setSelectedGroupId] = useState('');
      const [newStudentName, setNewStudentName] = useState('');
      const [newCatName, setNewCatName] = useState('');
      const [newCatType, setNewCatType] = useState('good');
      const [newCatPoints, setNewCatPoints] = useState(1);
      const [selectedStudentId, setSelectedStudentId] = useState('');
      const [selectedCatId, setSelectedCatId] = useState('');
      const [logNote, setLogNote] = useState('');

      const [batchNames, setBatchNames] = useState('');
      const [batchCategories, setBatchCategories] = useState('');

      // Audio / Noise Detector States
      const [isVoiceEnabled, setIsVoiceEnabled] = useState(false);
      const [volumeLevel, setVolumeLevel] = useState(0);
      const noiseThreshold = 60;
      const [isNoiseAlert, setIsNoiseAlert] = useState(false);

      const audioContextRef = useRef(null);
      const micStreamRef = useRef(null);
      const animFrameRef = useRef(null);

      // Current Active Class Data
      const activeClass = classesData[selectedClassId] || { name: 'Class', groups: [], logs: [] };

      // --- SAVE TO LOCALSTORAGE ---
      useEffect(() => {
        localStorage.setItem('GCM_CLASSES_DATA', JSON.stringify(classesData));
      }, [classesData]);

      useEffect(() => {
        localStorage.setItem('GCM_CATEGORIES', JSON.stringify(categories));
      }, [categories]);

      // Set default target group when switching class
      useEffect(() => {
        if (activeClass.groups.length > 0) {
          setSelectedGroupId(activeClass.groups[0].id);
        } else {
          setSelectedGroupId('');
        }
      }, [selectedClassId, classesData]);

      // --- CHECK TIME WINDOW (Weekdays 08:00 - 17:00) ---
      useEffect(() => {
        const checkTimeWindow = () => {
          const now = new Date();
          const day = now.getDay();    // 0 = Sun, 1 = Mon, ..., 5 = Fri, 6 = Sat
          const hour = now.getHours();  // 0 to 23

          const isWeekday = (day >= 1 && day <= 5);
          const isWithinHours = (hour >= START_HOUR && hour < END_HOUR);

          if (isWeekday && isWithinHours) {
            setIsTimeAllowed(true);
            setTimeStatusMessage('🟢 Editing Window OPEN (Weekdays 08:00 - 17:00)');
          } else {
            setIsTimeAllowed(false);
            setTimeStatusMessage('🔴 Editing Window CLOSED (Mon–Fri 08:00 - 17:00 Only)');
          }
        };

        checkTimeWindow();
        const interval = setInterval(checkTimeWindow, 30000);
        return () => clearInterval(interval);
      }, []);

      const handleLogin = () => {
        if (inputPassword === MASTER_PASSWORD) {
          setIsAuthenticated(true);
        } else {
          alert('Incorrect Password!');
        }
      };

      const canEdit = isAuthenticated && isTimeAllowed;

      // Audio Noise Detector
      useEffect(() => {
        if (isVoiceEnabled) startAudioMonitoring();
        else stopAudioMonitoring();
        return () => stopAudioMonitoring();
      }, [isVoiceEnabled]);

      const startAudioMonitoring = async () => {
        try {
          const stream = await navigator.mediaDevices.getUserMedia({ audio: true });
          micStreamRef.current = stream;
          const AudioCtx = window.AudioContext || window.webkitAudioContext;
          const audioCtx = new AudioCtx();
          audioContextRef.current = audioCtx;
          const analyser = audioCtx.createAnalyser();
          analyser.fftSize = 256;
          const source = audioCtx.createMediaStreamSource(stream);
          source.connect(analyser);

          const dataArray = new Uint8Array(analyser.frequencyBinCount);
          const updateVolume = () => {
            analyser.getByteFrequencyData(dataArray);
            let sum = 0;
            for (let i = 0; i < dataArray.length; i++) sum += dataArray[i];
            const average = sum / dataArray.length;
            const normalizedVol = Math.min(100, Math.round((average / 128) * 100));
            setVolumeLevel(normalizedVol);
            setIsNoiseAlert(normalizedVol > noiseThreshold);
            animFrameRef.current = requestAnimationFrame(updateVolume);
          };
          updateVolume();
        } catch (err) {
          alert("Microphone permission denied or blocked.");
          setIsVoiceEnabled(false);
        }
      };

      const stopAudioMonitoring = () => {
        if (animFrameRef.current) cancelAnimationFrame(animFrameRef.current);
        if (micStreamRef.current) micStreamRef.current.getTracks().forEach(track => track.stop());
        if (audioContextRef.current) audioContextRef.current.close();
        setVolumeLevel(0);
        setIsNoiseAlert(false);
      };

      // Actions for current active class
      const addGroup = () => {
        if (!canEdit) return alert('Editing is disabled!');
        if (!newGroupName.trim()) return;
        
        const newGroup = { id: `g_${Date.now()}`, name: newGroupName, members: [] };
        setClassesData({
          ...classesData,
          [selectedClassId]: {
            ...activeClass,
            groups: [...activeClass.groups, newGroup]
          }
        });
        setNewGroupName('');
      };

      const addStudent = () => {
        if (!canEdit) return alert('Editing is disabled!');
        if (!newStudentName.trim() || !selectedGroupId) return;

        const updatedGroups = activeClass.groups.map(g => g.id === selectedGroupId ? {
          ...g, members: [...g.members, { id: `s_${Date.now()}`, name: newStudentName, points: 0 }]
        } : g);

        setClassesData({
          ...classesData,
          [selectedClassId]: { ...activeClass, groups: updatedGroups }
        });
        setNewStudentName('');
      };

      const addCategory = () => {
        if (!canEdit) return alert('Editing is disabled!');
        if (!newCatName.trim()) return;
        const pts = newCatType === 'bad' ? -Math.abs(Number(newCatPoints)) : Math.abs(Number(newCatPoints));
        setCategories([...categories, { id: `c_${Date.now()}`, name: newCatName, type: newCatType, defaultPoints: pts }]);
        setNewCatName('');
      };

      const handleFastPasteStudents = () => {
        if (!canEdit) return alert('Editing is disabled!');
        if (!batchNames.trim() || !selectedGroupId) return;
        const namesArray = batchNames.split(/[\n,]+/).map(n => n.trim()).filter(n => n.length > 0);
        const newMembers = namesArray.map((name, index) => ({ id: `s_${Date.now()}_${index}`, name, points: 0 }));

        const updatedGroups = activeClass.groups.map(g => g.id === selectedGroupId ? {
          ...g, members: [...g.members, ...newMembers]
        } : g);

        setClassesData({
          ...classesData,
          [selectedClassId]: { ...activeClass, groups: updatedGroups }
        });
        setBatchNames('');
      };

      const handleFastPasteCategories = () => {
        if (!canEdit) return alert('Editing is disabled!');
        if (!batchCategories.trim()) return;
        const lines = batchCategories.split('\n').map(line => line.trim()).filter(line => line.length > 0);
        const newCats = [];
        lines.forEach((line, idx) => {
          const parts = line.split(',').map(p => p.trim());
          const name = parts[0];
          const type = parts[1] && parts[1].toLowerCase() === 'bad' ? 'bad' : 'good';
          let points = parts[2] ? Number(parts[2]) : (type === 'good' ? 1 : -1);
          if (type === 'bad' && points > 0) points = -points;
          if (type === 'good' && points < 0) points = Math.abs(points);
          if (name) newCats.push({ id: `c_${Date.now()}_${idx}`, name, type, defaultPoints: points });
        });
        setCategories([...categories, ...newCats]);
        setBatchCategories('');
      };

      const applyBehavior = () => {
        if (!canEdit) return alert('Editing is disabled!');
        if (!selectedStudentId || !selectedCatId) return;
        const cat = categories.find(c => c.id === selectedCatId);
        let targetStudentName = '';

        const updatedGroups = activeClass.groups.map(g => ({
          ...g,
          members: g.members.map(s => {
            if (s.id === selectedStudentId) {
              targetStudentName = s.name;
              return { ...s, points: s.points + cat.defaultPoints };
            }
            return s;
          })
        }));

        const newLog = {
          id: Date.now(),
          studentName: targetStudentName,
          categoryName: cat.name,
          points: cat.defaultPoints,
          type: cat.type,
          note: logNote,
          timestamp: new Date().toLocaleTimeString([], { hour: '2-digit', minute: '2-digit' })
        };

        setClassesData({
          ...classesData,
          [selectedClassId]: {
            ...activeClass,
            groups: updatedGroups,
            logs: [newLog, ...(activeClass.logs || [])]
          }
        });
        setLogNote('');
      };

      const resetCurrentClassData = () => {
        if (!canEdit) return alert('Editing is disabled!');
        if (confirm(`Are you sure you want to reset all data for ${activeClass.name}?`)) {
          setClassesData({
            ...classesData,
            [selectedClassId]: { name: activeClass.name, groups: [], logs: [] }
          });
        }
      };

      return (
        <div className="min-h-screen p-6 text-slate-800">
          {/* TOP BAR: CLASS SELECTOR & NOISE DETECTOR */}
          <header className="bg-white rounded-xl shadow-sm p-6 mb-4 border border-slate-200 flex justify-between items-center flex-wrap gap-4">
            <div className="flex items-center gap-4">
              <div>
                <label className="text-xs font-bold text-slate-400 uppercase tracking-wider block">Active Class</label>
                <select 
                  value={selectedClassId} 
                  onChange={(e) => setSelectedClassId(e.target.value)}
                  className="text-xl font-bold bg-slate-100 text-indigo-900 border border-slate-300 rounded-lg px-3 py-1.5 mt-0.5 focus:ring-2 focus:ring-indigo-500"
                >
                  {Object.keys(classesData).map(cid => (
                    <option key={cid} value={cid}>{classesData[cid].name}</option>
                  ))}
                </select>
              </div>
              <button 
                onClick={resetCurrentClassData} 
                className="text-xs bg-rose-50 text-rose-600 hover:bg-rose-100 border border-rose-200 px-2.5 py-1.5 rounded-lg self-end"
                title="Reset this class's roster & points"
              >
                Reset Class
              </button>
            </div>

            {/* Noise Detector */}
            <div className="flex items-center gap-4 bg-slate-100 p-3 rounded-lg border border-slate-200">
              <span className="text-sm font-semibold">Noise Detector</span>
              <label className="relative inline-flex items-center cursor-pointer">
                <input type="checkbox" checked={isVoiceEnabled} onChange={(e) => setIsVoiceEnabled(e.target.checked)} className="sr-only peer" />
                <div className="w-11 h-6 bg-slate-300 rounded-full peer peer-checked:after:translate-x-full peer-checked:after:border-white after:content-[''] after:absolute after:top-[2px] after:left-[2px] after:bg-white after:border-slate-300 after:border after:rounded-full after:h-5 after:w-5 after:transition-all peer-checked:bg-indigo-600"></div>
              </label>
              {isVoiceEnabled && (
                <div className="flex items-center gap-2 pl-2 border-l border-slate-300">
                  <div className="w-20 bg-slate-200 h-3 rounded-full overflow-hidden">
                    <div className={`h-full ${isNoiseAlert ? 'bg-red-500' : 'bg-emerald-500'}`} style={{ width: `${volumeLevel}%` }}></div>
                  </div>
                  <span className={`text-xs font-bold px-2 py-0.5 rounded ${isNoiseAlert ? 'bg-red-100 text-red-700 animate-bounce' : 'text-slate-600'}`}>
                    {isNoiseAlert ? 'TOO NOISY!' : `${volumeLevel}%`}
                  </span>
                </div>
              )}
            </div>
          </header>

          {/* AUTHORIZATION & TIME BAR */}
          <div className="bg-slate-900 text-white rounded-xl p-4 mb-6 shadow-md flex justify-between items-center flex-wrap gap-4">
            <div className="text-xs font-mono">
              <p className="font-bold">{timeStatusMessage}</p>
              <p className="text-slate-400 mt-0.5">
                Status: {canEdit ? '🔓 FULL EDITING ACCESS ACTIVE' : '🔒 VIEW-ONLY MODE'}
              </p>
            </div>

            {!isAuthenticated ? (
              <div className="flex gap-2">
                <input 
                  type="password" 
                  placeholder="Teacher Password" 
                  value={inputPassword} 
                  onChange={(e) => setInputPassword(e.target.value)} 
                  className="px-3 py-1.5 rounded text-xs text-slate-900 focus:outline-none"
                />
                <button onClick={handleLogin} className="bg-indigo-600 hover:bg-indigo-500 text-white px-3 py-1.5 rounded text-xs font-semibold">
                  Unlock
                </button>
              </div>
            ) : (
              <button onClick={() => setIsAuthenticated(false)} className="bg-rose-600 hover:bg-rose-500 text-white px-3 py-1.5 rounded text-xs font-semibold">
                Lock App
              </button>
            )}
          </div>

          {/* FAST PASTE SECTION */}
          <section className={`bg-indigo-50 border border-indigo-100 rounded-xl p-5 mb-6 ${!canEdit && 'opacity-60 pointer-events-none'}`}>
            <h2 className="text-md font-bold text-indigo-900 mb-3">⚡ Fast Paste / Batch Import ({activeClass.name})</h2>
            <div className="grid grid-cols-1 md:grid-cols-2 gap-4">
              <div className="bg-white p-4 rounded-lg border border-indigo-100 shadow-sm space-y-2">
                <label className="text-xs font-bold text-slate-700 block">Fast Paste Students (Line by line or comma-separated)</label>
                <div className="flex gap-2">
                  <span className="text-xs text-slate-500 self-center">Target Group:</span>
                  <select value={selectedGroupId} onChange={(e) => setSelectedGroupId(e.target.value)} className="border rounded px-2 py-1 text-xs bg-white">
                    {activeClass.groups.map(g => <option key={g.id} value={g.id}>{g.name}</option>)}
                  </select>
                </div>
                <textarea rows="3" placeholder="John Doe&#10;Jane Smith" value={batchNames} onChange={(e) => setBatchNames(e.target.value)} className="w-full border rounded p-2 text-xs focus:ring-1 focus:ring-indigo-500" />
                <button onClick={handleFastPasteStudents} disabled={!canEdit || activeClass.groups.length === 0} className="w-full bg-indigo-600 hover:bg-indigo-700 text-white text-xs font-semibold py-1.5 rounded disabled:bg-slate-400">Import All Students</button>
              </div>

              <div className="bg-white p-4 rounded-lg border border-indigo-100 shadow-sm space-y-2">
                <label className="text-xs font-bold text-slate-700 block">Fast Paste Categories (Format: Name, type, points)</label>
                <textarea rows="4" placeholder="Active Listening, good, 2&#10;Interrupting Class, bad, -2" value={batchCategories} onChange={(e) => setBatchCategories(e.target.value)} className="w-full border rounded p-2 text-xs focus:ring-1 focus:ring-indigo-500" />
                <button onClick={handleFastPasteCategories} disabled={!canEdit} className="w-full bg-indigo-600 hover:bg-indigo-700 text-white text-xs font-semibold py-1.5 rounded disabled:bg-slate-400">Import All Categories</button>
              </div>
            </div>
          </section>

          {/* MAIN GRID */}
          <div className="grid grid-cols-1 lg:grid-cols-3 gap-6">
            {/* GROUPS FOR ACTIVE CLASS */}
            <div className="space-y-6">
              <div className={`bg-white p-5 rounded-xl border border-slate-200 shadow-sm space-y-4 ${!canEdit && 'opacity-60'}`}>
                <h2 className="text-lg font-bold">Manage Groups ({activeClass.name})</h2>
                <div className="flex gap-2">
                  <input type="text" placeholder="New Group Name..." value={newGroupName} onChange={(e) => setNewGroupName(e.target.value)} className="flex-1 px-3 py-2 border rounded-lg text-sm" />
                  <button onClick={addGroup} disabled={!canEdit} className="bg-indigo-600 text-white px-3 py-2 rounded-lg text-sm disabled:bg-slate-400">+ Group</button>
                </div>
                <div className="flex gap-2">
                  <select value={selectedGroupId} onChange={(e) => setSelectedGroupId(e.target.value)} className="border rounded-lg px-2 py-2 text-sm bg-white">
                    {activeClass.groups.map(g => <option key={g.id} value={g.id}>{g.name}</option>)}
                  </select>
                  <input type="text" placeholder="Single Student Name..." value={newStudentName} onChange={(e) => setNewStudentName(e.target.value)} className="flex-1 px-3 py-2 border rounded-lg text-sm" />
                  <button onClick={addStudent} disabled={!canEdit || activeClass.groups.length === 0} className="bg-slate-800 text-white px-3 py-2 rounded-lg text-sm disabled:bg-slate-400">+ Student</button>
                </div>
              </div>

              {activeClass.groups.map(group => (
                <div key={group.id} className="bg-white rounded-xl border border-slate-200 p-4 shadow-sm">
                  <h3 className="font-bold border-b pb-2 mb-3 flex justify-between">
                    <span>{group.name}</span>
                    <span className="text-xs bg-slate-100 text-slate-600 px-2 py-1 rounded-full">{group.members.length} Members</span>
                  </h3>
                  <div className="space-y-2">
                    {group.members.map(s => (
                      <div key={s.id} className="flex justify-between items-center bg-slate-50 p-2.5 rounded-lg border border-slate-100">
                        <span className="font-medium text-sm">{s.name}</span>
                        <span className={`text-sm font-bold px-2 py-0.5 rounded ${s.points >= 0 ? 'bg-emerald-100 text-emerald-700' : 'bg-rose-100 text-rose-700'}`}>
                          {s.points > 0 ? `+${s.points}` : s.points} pts
                        </span>
                      </div>
                    ))}
                    {group.members.length === 0 && <p className="text-xs text-slate-400 italic">No members in this group yet.</p>}
                  </div>
                </div>
              ))}
              {activeClass.groups.length === 0 && (
                <div className="bg-white p-6 rounded-xl border border-slate-200 text-center text-slate-400 text-sm">
                  No groups created for {activeClass.name} yet. Add a group above!
                </div>
              )}
            </div>

            {/* BEHAVIOR LOGGING */}
            <div className="space-y-6">
              <div className={`bg-white p-5 rounded-xl border border-slate-200 shadow-sm space-y-4 ${!canEdit && 'opacity-60'}`}>
                <h2 className="text-lg font-bold">Log Behavior / Points</h2>
                <div>
                  <label className="text-xs font-semibold text-slate-500 uppercase">Select Student ({activeClass.name})</label>
                  <select value={selectedStudentId} onChange={(e) => setSelectedStudentId(e.target.value)} className="w-full mt-1 border rounded-lg p-2 text-sm bg-white">
                    <option value="">-- Choose Student --</option>
                    {activeClass.groups.flatMap(g => g.members).map(s => (
                      <option key={s.id} value={s.id}>{s.name}</option>
                    ))}
                  </select>
                </div>
                <div>
                  <label className="text-xs font-semibold text-slate-500 uppercase">Behavior Category</label>
                  <select value={selectedCatId} onChange={(e) => setSelectedCatId(e.target.value)} className="w-full mt-1 border rounded-lg p-2 text-sm bg-white">
                    <option value="">-- Select Category --</option>
                    {categories.map(c => (
                      <option key={c.id} value={c.id}>[{c.type.toUpperCase()}] {c.name} ({c.defaultPoints > 0 ? `+${c.defaultPoints}` : c.defaultPoints} pts)</option>
                    ))}
                  </select>
                </div>
                <div>
                  <label className="text-xs font-semibold text-slate-500 uppercase">Suggestion / Notes</label>
                  <textarea rows="2" placeholder="e.g., Interrupting group discussion..." value={logNote} onChange={(e) => setLogNote(e.target.value)} className="w-full mt-1 border rounded-lg p-2 text-sm" />
                </div>
                <button onClick={applyBehavior} disabled={!canEdit} className="w-full bg-indigo-600 text-white font-medium py-2 rounded-lg text-sm disabled:bg-slate-400">Submit Behavior Entry</button>
              </div>

              <div className={`bg-white p-5 rounded-xl border border-slate-200 shadow-sm space-y-4 ${!canEdit && 'opacity-60'}`}>
                <h2 className="text-lg font-bold">Add Single Category</h2>
                <div className="grid grid-cols-2 gap-2">
                  <input type="text" placeholder="Category Name" value={newCatName} onChange={(e) => setNewCatName(e.target.value)} className="col-span-2 border rounded-lg p-2 text-sm" />
                  <select value={newCatType} onChange={(e) => setNewCatType(e.target.value)} className="border rounded-lg p-2 text-sm bg-white">
                    <option value="good">Good (Reward)</option>
                    <option value="bad">Bad (Punishment)</option>
                  </select>
                  <input type="number" placeholder="Points" value={newCatPoints} onChange={(e) => setNewCatPoints(e.target.value)} className="border rounded-lg p-2 text-sm" />
                </div>
                <button onClick={addCategory} disabled={!canEdit} className="w-full bg-slate-800 text-white text-sm py-2 rounded-lg disabled:bg-slate-400">Add Category</button>
              </div>
            </div>

            {/* ACTIVITY LOG */}
            <div className="bg-white p-5 rounded-xl border border-slate-200 shadow-sm h-fit">
              <h2 className="text-lg font-bold mb-4">Activity Log ({activeClass.name})</h2>
              <div className="space-y-3 max-h-[500px] overflow-y-auto">
                {(activeClass.logs || []).map(log => (
                  <div key={log.id} className="p-3 border rounded-lg bg-slate-50 text-sm space-y-1">
                    <div className="flex justify-between items-center">
                      <span className="font-bold">{log.studentName}</span>
                      <span className="text-xs text-slate-400">{log.timestamp}</span>
                    </div>
                    <div className="flex justify-between items-center">
                      <span className="text-slate-600">{log.categoryName}</span>
                      <span className={`font-bold ${log.type === 'good' ? 'text-emerald-600' : 'text-rose-600'}`}>
                        {log.points > 0 ? `+${log.points}` : log.points} pts
                      </span>
                    </div>
                    {log.note && <p className="text-xs text-slate-500 italic bg-white p-1.5 rounded border border-slate-100">{log.note}</p>}
                  </div>
                ))}
                {(!activeClass.logs || activeClass.logs.length === 0) && (
                  <p className="text-sm text-slate-400 text-center py-6">No behavioral logs recorded for {activeClass.name} yet.</p>
                )}
              </div>
            </div>
          </div>
        </div>
      );
    }

    ReactDOM.createRoot(document.getElementById('root')).render(<GroupClassroomManager />);
  </script>
</body>
</html>
