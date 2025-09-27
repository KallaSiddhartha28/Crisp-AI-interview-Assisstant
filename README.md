import React, { useState, useEffect, useCallback } from 'react';
// Assume Tailwind CSS and lucide-react (using inline SVG for lucide icons) are available

// --- CONFIG AND DATA ---
const APP_NAME = "Crisp AI Interview Assistant";

const INTERVIEW_QUESTIONS = [
  { id: 1, difficulty: 'Easy', timeLimit: 20, text: "Explain the difference between state and props in React." },
  { id: 2, difficulty: 'Easy', timeLimit: 20, text: "What is event bubbling, and how do you prevent it in the DOM?" },
  { id: 3, difficulty: 'Medium', timeLimit: 60, text: "Describe a common problem solved by React Hooks, and show an example of a custom hook you might write." },
  { id: 4, difficulty: 'Medium', timeLimit: 60, text: "Explain the concept of middleware in an Express.js (Node) application and give two examples." },
  { id: 5, difficulty: 'Hard', timeLimit: 120, text: "How would you architect a GraphQL server with Node.js, specifically focusing on data loaders to solve the N+1 problem?" },
  { id: 6, difficulty: 'Hard', timeLimit: 120, text: "Design a responsive, high-performance feed component in React. Discuss virtualization strategies and how you would handle infinite scrolling efficiently." },
];

const INITIAL_SESSION_STATE = {
    id: crypto.randomUUID(), // Unique ID for the current session/candidate
    profile: { name: '', email: '', phone: '' },
    chatHistory: [],
    currentQuestionIndex: -1,
    timer: 0,
    isTimerActive: false,
    interviewPhase: 'setup', // 'setup', 'collecting_info', 'running', 'complete'
    selectedFile: null,
    isProcessing: false,
    input: '', // Dedicated input state for persistence
    missingField: null,
};

// --- UTILITY HOOK FOR LOCAL STORAGE PERSISTENCE ---
const useLocalStorage = (key, initialValue) => {
    const [storedValue, setStoredValue] = useState(() => {
        try {
            const item = window.localStorage.getItem(key);
            return item ? JSON.parse(item) : initialValue;
        } catch (error) {
            console.error('Error reading localStorage key “' + key + '”: ', error);
            return initialValue;
        }
    });

    const setValue = (value) => {
        try {
            const valueToStore = value instanceof Function ? value(storedValue) : value;
            setStoredValue(valueToStore);
            window.localStorage.setItem(key, JSON.stringify(valueToStore));
        } catch (error) {
            console.error('Error writing localStorage key “' + key + '”: ', error);
        }
    };

    return [storedValue, setValue];
};

const createUniqueId = () => crypto.randomUUID();

// --- INLINE SVG ICONS (for a single-file React component) ---
const FileTextIcon = (props) => (<svg {...props} xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round" strokeLinejoin="round"><path d="M15 2H6a2 2 0 0 0-2 2v16a2 2 0 0 0 2 2h12a2 2 0 0 0 2-2V7Z"/><path d="M14 2v4a2 2 0 0 0 2 2h4"/><line x1="16" x2="8" y1="13" y2="13"/><line x1="16" x2="8" y1="17" y2="17"/><line x1="10" x2="8" y1="9" y2="9"/></svg>);
const UserIcon = (props) => (<svg {...props} xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round" strokeLinejoin="round"><path d="M19 21v-2a4 4 0 0 0-4-4H9a4 4 0 0 0-4 4v2"/><circle cx="12" cy="7" r="4"/></svg>);
const SendIcon = (props) => (<svg {...props} xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round" strokeLinejoin="round"><path d="m22 2-7 20-4-9-9-4Z"/><path d="M22 2 11 13"/></svg>);
const UploadIcon = (props) => (<svg {...props} xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round" strokeLinejoin="round"><path d="M21 15v4a2 2 0 0 1-2 2H5a2 2 0 0 1-2-2v-4"/><polyline points="17 8 12 3 7 8"/><line x1="12" x2="12" y1="3" y2="15"/></svg>);
const ZapIcon = (props) => (<svg {...props} xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round" strokeLinejoin="round"><polygon points="13 2 3 14 12 14 11 22 21 10 12 10 13 2"/></svg>);
const LoaderIcon = () => (<svg className="animate-spin -ml-1 mr-3 h-5 w-5 text-white" xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24"><circle className="opacity-25" cx="12" cy="12" r="10" stroke="currentColor" strokeWidth="4"></circle><path className="opacity-75" fill="currentColor" d="M4 12a8 8 0 018-8V0C5.373 0 0 5.373 0 12h4zm2 5.291A7.962 7.962 0 014 12H0c0 3.042 1.135 5.824 3 7.938l3-2.647z"></path></svg>);
const LayoutGridIcon = (props) => (<svg {...props} xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round" strokeLinejoin="round"><rect width="7" height="7" x="3" y="3" rx="1"/><rect width="7" height="7" x="14" y="3" rx="1"/><rect width="7" height="7" x="14" y="14" rx="1"/><rect width="7" height="7" x="3" y="14" rx="1"/></svg>);
const MessageCircleIcon = (props) => (<svg {...props} xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round" strokeLinejoin="round"><path d="M7.9 20A9 9 0 1 0 4 16.1L2 22Z"/></svg>);


// --- SHARED COMPONENTS ---

const ChatMessage = React.memo(({ message }) => {
  const isUser = message.role === 'user';
  const isSystem = message.role === 'system';
  
  const bgColor = isUser ? 'bg-indigo-50 shadow-md shadow-indigo-200/50' : 'bg-white shadow-lg';
  const alignment = isUser ? 'justify-end' : 'justify-start';
  const systemBorder = isSystem ? 'border-l-4 border-indigo-500 pl-3' : '';

  return (
    <div className={`flex ${alignment} my-3`}>
      <div 
        className={`max-w-xs md:max-w-md lg:max-w-lg p-3 rounded-xl ${bgColor} ${systemBorder}
                    ${isUser ? 'rounded-br-none' : 'rounded-tl-none'} transition-all duration-300 transform hover:scale-[1.01]`}
      >
        <p className="text-sm font-medium text-gray-800 whitespace-pre-wrap">{message.text}</p>
        <p className={`text-xs mt-1 ${isUser ? 'text-indigo-500 text-right' : 'text-gray-400 text-left'}`}>
          {new Date(message.timestamp).toLocaleTimeString()}
        </p>
      </div>
    </div>
  );
});

const InterviewStatus = ({ phase }) => {
    let color, text, icon;

    switch (phase) {
        case 'setup':
            color = 'bg-yellow-100 text-yellow-800 border-yellow-300';
            text = 'Setup: Resume Required';
            icon = <FileTextIcon className="w-4 h-4 mr-2" />;
            break;
        case 'collecting_info':
            color = 'bg-orange-100 text-orange-800 border-orange-300';
            text = 'Profile Information Pending';
            icon = <UserIcon className="w-4 h-4 mr-2" />;
            break;
        case 'running':
            color = 'bg-green-100 text-green-800 border-green-300';
            text = 'Interview Running';
            icon = <ZapIcon className="w-4 h-4 mr-2" />;
            break;
        case 'complete':
            color = 'bg-blue-100 text-blue-800 border-blue-300';
            text = 'Interview Complete (Score Finalized)';
            icon = <FileTextIcon className="w-4 h-4 mr-2" />;
            break;
        case 'abandoned':
            color = 'bg-red-100 text-red-800 border-red-300';
            text = 'Session Abandoned';
            icon = <ZapIcon className="w-4 h-4 mr-2" />;
            break;
        default:
            color = 'bg-gray-100 text-gray-600 border-gray-300';
            text = 'Initializing...';
            icon = null;
    }

    return (
        <div className={`flex items-center justify-center p-2 rounded-lg border font-medium text-sm transition-colors duration-300 ${color} mb-4`}>
            {icon}
            {text}
        </div>
    );
};

const TimerDisplay = React.memo(({ time, difficulty, totalTime }) => {
    const minutes = Math.floor(time / 60);
    const seconds = time % 60;
    const formattedTime = `${minutes.toString().padStart(2, '0')}:${seconds.toString().padStart(2, '0')}`;
    
    let colorClass = 'from-blue-600 to-blue-500';
    let pulse = '';
    
    if (time < totalTime * 0.25) { 
        colorClass = 'from-red-600 to-red-500'; 
        pulse = 'animate-pulse';
    } else if (time < totalTime * 0.5) { 
         colorClass = 'from-yellow-600 to-yellow-500';
    }

    return (
        <div className={`flex items-center justify-between p-3 rounded-xl text-white font-bold shadow-lg transition-all duration-300 bg-gradient-to-r ${colorClass} ${pulse} mb-4`}>
            <div className="flex items-center">
                <ZapIcon className="w-5 h-5 mr-2" />
                <span>{difficulty} Question</span>
            </div>
            <div className="text-2xl font-mono border-l border-white/50 pl-3">
                <span className="tabular-nums">{formattedTime}</span>
            </div>
        </div>
    );
});

// --- INTERVIEWER DASHBOARD COMPONENT ---

const InterviewerDashboard = ({ allCandidates, setCurrentSession, setActiveTab }) => {
    const [searchTerm, setSearchTerm] = useState('');
    const [sortKey, setSortKey] = useState('timestamp');
    const [sortDirection, setSortDirection] = useState('desc');
    const [selectedCandidate, setSelectedCandidate] = useState(null);

    const sortCandidates = (a, b) => {
        let valA, valB;

        switch (sortKey) {
            case 'score':
                // Treat 'N/A' as 0 for sorting
                valA = parseInt(a.score) || 0;
                valB = parseInt(b.score) || 0;
                break;
            case 'profile.name':
                valA = a.profile.name.toLowerCase();
                valB = b.profile.name.toLowerCase();
                break;
            default: // 'timestamp'
                valA = a.timestamp;
                valB = b.timestamp;
        }

        if (valA < valB) return sortDirection === 'asc' ? -1 : 1;
        if (valA > valB) return sortDirection === 'asc' ? 1 : -1;
        return 0;
    };

    const filteredCandidates = allCandidates
        .filter(c => c.profile.name.toLowerCase().includes(searchTerm.toLowerCase()) || c.id.includes(searchTerm))
        .sort(sortCandidates);

    if (selectedCandidate) {
        return (
            <div className="p-6 bg-white rounded-xl shadow-xl border border-gray-100">
                <button 
                    onClick={() => setSelectedCandidate(null)}
                    className="mb-4 text-indigo-600 hover:text-indigo-800 font-medium transition duration-150 flex items-center"
                >
                    &larr; Back to Candidate List
                </button>
                <h2 className="text-3xl font-bold text-gray-800 mb-4">{selectedCandidate.profile.name || 'Anonymous Candidate'}</h2>
                
                <div className="grid grid-cols-1 md:grid-cols-3 gap-4 mb-6 text-center">
                    <div className="p-4 bg-indigo-50 rounded-lg">
                        <p className="text-sm font-semibold text-indigo-700">Final Score</p>
                        <p className="text-3xl font-extrabold text-indigo-600">{selectedCandidate.score || 'N/A'}</p>
                    </div>
                    <div className="p-4 bg-gray-50 rounded-lg">
                        <p className="text-sm font-semibold text-gray-700">Status</p>
                        <p className="text-xl font-bold text-green-600 capitalize">{selectedCandidate.interviewPhase}</p>
                    </div>
                     <div className="p-4 bg-gray-50 rounded-lg">
                        <p className="text-sm font-semibold text-gray-700">Email</p>
                        <p className="text-lg font-bold text-gray-700 truncate">{selectedCandidate.profile.email || 'N/A'}</p>
                    </div>
                </div>

                <div className="p-4 bg-white rounded-xl shadow-inner mb-6 border-b border-gray-200">
                    <p className="text-lg font-semibold text-gray-800 mb-2">AI Summary</p>
                    <p className="text-gray-600 italic whitespace-pre-wrap">{selectedCandidate.summary || 'Summary pending/interview incomplete.'}</p>
                </div>

                <h3 className="text-xl font-bold text-gray-800 mb-3">Interview Transcript</h3>
                <div className="max-h-96 overflow-y-auto border p-3 rounded-lg bg-gray-50">
                    {selectedCandidate.chatHistory.map((msg, index) => (
                        <div key={index} className={`mb-2 p-2 rounded-lg ${msg.role === 'user' ? 'bg-indigo-100' : 'bg-white border'}`}>
                            <span className={`font-bold ${msg.role === 'user' ? 'text-indigo-700' : 'text-gray-600'}`}>{msg.role === 'user' ? 'Candidate' : 'Crisp AI'}:</span>
                            <p className="text-sm mt-1 whitespace-pre-wrap">{msg.text}</p>
                        </div>
                    ))}
                </div>
            </div>
        );
    }

    return (
        <div className="p-6">
            <h2 className="text-2xl font-bold text-gray-800 mb-6">Candidate Dashboard</h2>
            <div className="flex flex-col sm:flex-row space-y-3 sm:space-y-0 sm:space-x-4 mb-6">
                <input
                    type="text"
                    placeholder="Search by name or ID..."
                    value={searchTerm}
                    onChange={(e) => setSearchTerm(e.target.value)}
                    className="p-3 border border-gray-300 rounded-xl focus:ring-indigo-500 focus:border-indigo-500 transition duration-150 shadow-sm flex-1"
                />
                <select
                    value={sortKey}
                    onChange={(e) => setSortKey(e.target.value)}
                    className="p-3 border border-gray-300 rounded-xl shadow-sm bg-white"
                >
                    <option value="timestamp">Sort by Date</option>
                    <option value="score">Sort by Score</option>
                    <option value="profile.name">Sort by Name</option>
                </select>
                <select
                    value={sortDirection}
                    onChange={(e) => setSortDirection(e.target.value)}
                    className="p-3 border border-gray-300 rounded-xl shadow-sm bg-white"
                >
                    <option value="desc">Descending</option>
                    <option value="asc">Ascending</option>
                </select>
            </div>
            
            <div className="overflow-x-auto bg-white rounded-xl shadow-xl">
                <table className="min-w-full divide-y divide-gray-200">
                    <thead className="bg-gray-50">
                        <tr>
                            <th className="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">Candidate Name</th>
                            <th className="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">Status</th>
                            <th className="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">Score</th>
                            <th className="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">Actions</th>
                        </tr>
                    </thead>
                    <tbody className="bg-white divide-y divide-gray-200">
                        {filteredCandidates.map((candidate) => (
                            <tr key={candidate.id} className="hover:bg-indigo-50 transition duration-100">
                                <td className="px-6 py-4 whitespace-nowrap text-sm font-medium text-gray-900 truncate max-w-xs">{candidate.profile.name || `Anon Candidate (${candidate.id.substring(0, 8)}...)`}</td>
                                <td className="px-6 py-4 whitespace-nowrap text-sm capitalize">
                                    <span className={`px-2 inline-flex text-xs leading-5 font-semibold rounded-full 
                                        ${candidate.interviewPhase === 'complete' ? 'bg-green-100 text-green-800' : 
                                          (candidate.interviewPhase === 'abandoned' ? 'bg-red-100 text-red-800' : 'bg-yellow-100 text-yellow-800')
                                        }`}>
                                        {candidate.interviewPhase.replace('_', ' ')}
                                    </span>
                                </td>
                                <td className="px-6 py-4 whitespace-nowrap text-sm text-gray-700 font-bold">{candidate.score || 'N/A'}</td>
                                <td className="px-6 py-4 whitespace-nowrap text-sm font-medium">
                                    <button 
                                        onClick={() => setSelectedCandidate(candidate)}
                                        className="text-indigo-600 hover:text-indigo-900 mr-4"
                                    >
                                        View Details
                                    </button>
                                    {(candidate.interviewPhase !== 'complete' && candidate.id !== setCurrentSession.id) && (
                                        <button 
                                            onClick={() => {
                                                // Load this candidate's state into the current session
                                                setCurrentSession(candidate);
                                                setActiveTab('interviewee');
                                            }}
                                            className="text-blue-600 hover:text-blue-900"
                                        >
                                            Resume
                                        </button>
                                    )}
                                </td>
                            </tr>
                        ))}
                    </tbody>
                </table>
                {allCandidates.length === 0 && (
                    <div className="p-6 text-center text-gray-500">No interview data found.</div>
                )}
            </div>
        </div>
    );
};

// --- WELCOME BACK MODAL COMPONENT ---
const WelcomeBackModal = ({ session, onResume, onStartNew, onClose }) => {
    return (
        <div className="fixed inset-0 bg-black bg-opacity-50 flex items-center justify-center z-50 p-4">
            <div className="bg-white p-8 rounded-xl shadow-2xl max-w-sm w-full transform transition-all">
                <h3 className="text-2xl font-bold text-indigo-700 mb-3">Welcome Back to Crisp!</h3>
                <p className="text-gray-700 mb-5">
                    We detected an unfinished session for **{session.profile.name || 'an anonymous candidate'}**. Status: <span className="capitalize font-semibold">{session.interviewPhase.replace('_', ' ')}</span>.
                </p>
                
                <div className="space-y-3">
                    <button 
                        onClick={onResume}
                        className="w-full bg-green-500 hover:bg-green-600 text-white font-bold py-3 rounded-xl transition duration-200 shadow-lg"
                    >
                        Resume Session
                    </button>
                    <button 
                        onClick={onStartNew}
                        className="w-full bg-yellow-500 hover:bg-yellow-600 text-white font-bold py-3 rounded-xl transition duration-200 shadow-lg"
                    >
                        Start New Interview
                    </button>
                </div>
                <button onClick={onClose} className="mt-4 w-full text-sm text-gray-500 hover:text-gray-700">
                    Close (Will resume later)
                </button>
            </div>
        </div>
    );
};


// --- MAIN APP COMPONENT ---

export default function App() {
  const [activeTab, setActiveTab] = useState('interviewee');
  const [showWelcomeBackModal, setShowWelcomeBackModal] = useState(false);
    
  // State for all candidates (Dashboard data)
  const [allCandidates, setAllCandidates] = useLocalStorage('crisp_candidates', []);
    
  // State for the current active interview session (Interviewee chat data)
  const [session, setSession] = useLocalStorage('crisp_session', INITIAL_SESSION_STATE);
  
  // Destructure frequently used session fields for cleaner code
  const { 
    chatHistory, currentQuestionIndex, timer, isTimerActive, 
    interviewPhase, selectedFile, isProcessing, profile, input, missingField
  } = session;

  // --- PERSISTENCE AND RESUME LOGIC ---
  useEffect(() => {
    // Check for unfinished session on initial load
    if (session.interviewPhase !== 'setup' && session.interviewPhase !== 'complete') {
        setShowWelcomeBackModal(true);
    }
  }, []); // Run only on mount

  // --- Core Timer Logic ---
  const handleTimeOut = useCallback(() => {
    setSession(prev => {
        // Only run timeout logic if the timer was active
        if (!prev.isTimerActive || prev.timer !== 0) return prev;
        
        const question = INTERVIEW_QUESTIONS[prev.currentQuestionIndex];
        const nextIndex = prev.currentQuestionIndex + 1;
        const newHistory = [...prev.chatHistory, {
            id: createUniqueId(),
            role: 'system',
            text: `**[SYSTEM] Time's up!** Your answer for Q${question.id} (${question.difficulty}) was automatically submitted.`,
            timestamp: Date.now() + 10,
        }];

        if (nextIndex < INTERVIEW_QUESTIONS.length) {
            // Move to next question immediately
            const nextQuestion = INTERVIEW_QUESTIONS[nextIndex];
            newHistory.push({
                id: createUniqueId(),
                role: 'system',
                text: `**Q${nextQuestion.id} (${nextQuestion.difficulty}, ${nextQuestion.timeLimit}s):** ${nextQuestion.text}`,
                timestamp: Date.now() + 20,
            });

            return {
                ...prev,
                isTimerActive: true,
                currentQuestionIndex: nextIndex,
                timer: nextQuestion.timeLimit,
                chatHistory: newHistory,
            };

        } else {
            // End interview
            const finalSession = {
                ...prev,
                interviewPhase: 'complete',
                isTimerActive: false,
                chatHistory: [...newHistory, {
                    id: createUniqueId(),
                    role: 'system',
                    text: `**Interview Complete!** Thank you for participating. The AI is now calculating your final score and summary.`,
                    timestamp: Date.now() + 30,
                }],
                // Mock final score and summary for dashboard
                score: '72%', 
                summary: "Candidate timed out on the final hard question but showed promising foundational knowledge in easy and medium sections. Needs to improve speed and depth on architectural topics.",
            };
            saveCandidateToDashboard(finalSession);
            return INITIAL_SESSION_STATE; // Reset session after saving
        }
    });
  }, [setSession, allCandidates]); // Added allCandidates to dependencies to satisfy linter rule for saveCandidateToDashboard dependency

  // Timer loop effect
  useEffect(() => {
    let interval;
    if (isTimerActive && timer > 0) {
      interval = setInterval(() => {
        setSession(prev => ({ ...prev, timer: prev.timer - 1 }));
      }, 1000);
    } else if (timer === 0 && isTimerActive) {
      clearInterval(interval);
      handleTimeOut();
    }
    return () => clearInterval(interval);
  }, [isTimerActive, timer, handleTimeOut]);
    
  // Auto-scroll logic for chat
  useEffect(() => {
    const main = document.getElementById('chat-main');
    if (main) {
      main.scrollTop = main.scrollHeight;
    }
  }, [chatHistory]);


  // --- INTERVIEW FLOW HELPERS ---

  const saveCandidateToDashboard = useCallback((finalSession) => {
    // Create the final data object
    const candidateData = {
        id: finalSession.id,
        profile: finalSession.profile,
        score: finalSession.score || 'N/A', 
        summary: finalSession.summary || "Summary pending/interview incomplete.",
        interviewPhase: finalSession.interviewPhase,
        chatHistory: finalSession.chatHistory,
        timestamp: Date.now(),
    };

    setAllCandidates(prev => {
        // Replace existing entry (for resuming/abandoning) or add new one
        const existingIndex = prev.findIndex(c => c.id === candidateData.id);
        if (existingIndex > -1) {
            return prev.map((c, i) => i === existingIndex ? candidateData : c);
        }
        return [...prev, candidateData];
    });

  }, [setAllCandidates]);


  const startNextQuestion = useCallback((index) => {
      const nextQuestion = INTERVIEW_QUESTIONS[index];
      
      setSession(prev => ({
          ...prev,
          currentQuestionIndex: index,
          timer: nextQuestion.timeLimit,
          isTimerActive: true,
          interviewPhase: 'running',
          chatHistory: [...prev.chatHistory, {
              id: createUniqueId(),
              role: 'system',
              text: `**Q${nextQuestion.id} (${nextQuestion.difficulty}, ${nextQuestion.timeLimit}s):** ${nextQuestion.text}`,
              timestamp: Date.now() + 10,
          }]
      }));
  }, [setSession]);

  // --- RESUME PROCESSING & MISSING FIELD CHECK ---

  // Utility to find the next missing field prompt
  const getNextMissingFieldPrompt = (currentProfile) => {
    if (!currentProfile.name) {
        return { field: 'name', prompt: 'I could not confirm your full name from the resume. What is your full name?' };
    } 
    if (!currentProfile.email) {
        return { field: 'email', prompt: 'I could not confirm your email address. What is your preferred email?' };
    } 
    if (!currentProfile.phone) {
        return { field: 'phone', prompt: 'I could not confirm a phone number. What is your primary contact number?' };
    }
    return null; // No missing fields
  };


  const startInterview = useCallback(() => {
    if (interviewPhase !== 'setup' || isProcessing || !selectedFile) return;
    
    setSession(prev => ({ ...prev, isProcessing: true }));

    setTimeout(() => {
        // --- Mock Resume Extraction ---
        const extractedProfile = {
            name: 'Alex Johnson',
            email: 'alex.j@example.com',
            phone: '', // MOCK: Phone number is missing
        };

        const welcomeMessage = `Welcome to **Crisp**, ${extractedProfile.name || 'Candidate'}! I am your AI Interview Assistant. We are ready to begin the profile check.`;

        let initialChatHistory = [{ 
            id: createUniqueId(), 
            role: 'system', 
            text: `[FILE SYSTEM] Resume **${selectedFile.name}** processed.\n\n` + welcomeMessage,
            timestamp: Date.now() 
        }];
        
        const nextMissing = getNextMissingFieldPrompt(extractedProfile);

        if (nextMissing) {
            setSession(prev => ({
                ...prev,
                interviewPhase: 'collecting_info',
                missingField: nextMissing.field,
                profile: extractedProfile,
                isProcessing: false,
                chatHistory: [...initialChatHistory, {
                    id: createUniqueId(), 
                    role: 'system', 
                    text: `**Profile Check:** ${nextMissing.prompt}`,
                    timestamp: Date.now() + 10, 
                }],
            }));
        } else {
            // No missing fields, proceed directly to running
             setSession(prev => ({ ...prev, interviewPhase: 'running', isProcessing: false, profile: extractedProfile }));
             startNextQuestion(0);
        }

    }, 1500);
  }, [interviewPhase, isProcessing, selectedFile, setSession, startNextQuestion]);

  
  const handleSendMessage = (e) => {
    e.preventDefault();
    if (!input.trim() || isProcessing || interviewPhase === 'setup' || interviewPhase === 'complete') return;

    const userMessage = { id: createUniqueId(), role: 'user', text: input.trim(), timestamp: Date.now() };
    
    // Save user message and start processing state
    setSession(prev => ({ ...prev, chatHistory: [...prev.chatHistory, userMessage], input: '', isProcessing: true }));

    setTimeout(() => {
        let aiResponseText = '';
        let nextPhase = interviewPhase;
        let updatedProfile = { ...profile };

        if (interviewPhase === 'collecting_info') {
            const answer = userMessage.text;
            updatedProfile[missingField] = answer;

            aiResponseText = `Thank you. I have recorded your ${missingField} as: **${answer}**.`;
            
            const nextMissing = getNextMissingFieldPrompt(updatedProfile);

            if (nextMissing) {
                // More fields are missing
                aiResponseText += `\n\n**Profile Check:** ${nextMissing.prompt}`;
                setSession(prev => ({ ...prev, missingField: nextMissing.field, profile: updatedProfile, isProcessing: false, chatHistory: [...prev.chatHistory, { id: createUniqueId(), role: 'system', text: aiResponseText, timestamp: Date.now() + 20 }] }));
                return;
            } else {
                // All fields collected, start interview
                aiResponseText += '\n\n**All profile information collected.** Starting the timed interview now. Good luck!';
                nextPhase = 'running';
                
                setSession(prev => ({ 
                    ...prev, 
                    missingField: null, 
                    profile: updatedProfile, 
                    isProcessing: false, 
                    chatHistory: [...prev.chatHistory, { id: createUniqueId(), role: 'system', text: aiResponseText, timestamp: Date.now() + 20 }] 
                }));
                
                // Start the first question after AI response is logged
                setTimeout(() => startNextQuestion(0), 100); 
            }

        } else if (interviewPhase === 'running') {
            // Handle timed question answer
            const currentQ = INTERVIEW_QUESTIONS[currentQuestionIndex];
            const timeElapsed = currentQ.timeLimit - timer;
            const nextIndex = currentQuestionIndex + 1;
            
            // 1. Stop Timer
            setSession(prev => ({ ...prev, isTimerActive: false }));

            aiResponseText = `(AI Feedback for Q${currentQ.id}): Thank you. Time taken: ${timeElapsed} seconds. The AI is analyzing your response...`;

            if (nextIndex < INTERVIEW_QUESTIONS.length) {
                // 2. Move to next question
                setSession(prev => ({ ...prev, isProcessing: false, chatHistory: [...prev.chatHistory, { id: createUniqueId(), role: 'system', text: aiResponseText, timestamp: Date.now() + 20 }] }));
                setTimeout(() => startNextQuestion(nextIndex), 100); 
            } else {
                // 3. End Interview
                aiResponseText += `\n\n**Interview Complete!** The AI is calculating your final score and summary.`;
                const finalSession = {
                    ...session, 
                    interviewPhase: 'complete', 
                    isTimerActive: false, 
                    input: '',
                    // Mock final score and summary for dashboard
                    score: '85%', 
                    summary: "Candidate provided solid, confident answers on all 6 questions, demonstrating strong React state management skills and a clear understanding of Express middleware. Final answer on GraphQL scaling was conceptually strong.",
                    chatHistory: [...session.chatHistory, userMessage, { id: createUniqueId(), role: 'system', text: aiResponseText, timestamp: Date.now() + 20 }]
                };
                
                saveCandidateToDashboard(finalSession);
                setSession(INITIAL_SESSION_STATE); // Reset current session state
            }
        }
    }, 1000); // Simulated AI response delay
  };
  
  const handleFileChange = (event) => {
      const file = event.target.files[0];
      if (file) {
          setSession(prev => ({ 
              ...prev, 
              selectedFile: { name: file.name, type: file.type },
              chatHistory: [...prev.chatHistory, { id: createUniqueId(), role: 'system', text: `[FILE SYSTEM] Resume **${file.name}** selected. Click 'Start AI Interview' to process.`, timestamp: Date.now() }]
          }));
      }
  };

  // --- Resume and New Session Handlers ---
  const handleStartNewSession = () => {
      // Find the existing session and mark it as abandoned before starting new
      if (interviewPhase !== 'setup' && interviewPhase !== 'complete') {
          saveCandidateToDashboard({ 
              ...session, 
              score: 'N/A', 
              summary: 'Candidate abandoned session to start a new one.', 
              interviewPhase: 'abandoned',
              isTimerActive: false,
              input: '' // Clear input
          });
      }
      setSession(INITIAL_SESSION_STATE);
      setShowWelcomeBackModal(false);
  };
  
  const handleResumeSession = () => {
      setShowWelcomeBackModal(false);
      // Logic relies on useEffect(timer) to pick up and resume timing if isTimerActive is true
      setSession(prev => ({ ...prev, isProcessing: false })); // Ensure processing flag is off
  };

  const handleModalClose = () => {
      setShowWelcomeBackModal(false);
  }
  
  // --- SUB-COMPONENT RENDERERS ---

  const ResumeInputArea = () => (
    <div className="bg-white p-6 rounded-xl shadow-xl border border-gray-100 transition duration-500">
      <h2 className="text-xl font-bold text-gray-800 flex items-center mb-4">
        <FileTextIcon className="w-6 h-6 mr-3 text-indigo-600" />
        Step 1: Upload Your Resume
      </h2>
      <p className="text-sm text-gray-600 mb-6">
        Upload your resume (.pdf or .docx) to allow **Crisp AI** to tailor the interview.
      </p>

      {/* File Input and Selection Display */}
      <div className="flex flex-col sm:flex-row items-stretch sm:items-center space-y-3 sm:space-y-0 sm:space-x-3 mb-6">
        <label 
            htmlFor="resume-upload" 
            className={`cursor-pointer w-full sm:w-auto text-center font-medium py-3 px-6 rounded-xl shadow-lg transition duration-200 flex items-center justify-center 
                ${isProcessing ? 'bg-gray-400 text-gray-700' : 'bg-indigo-600 hover:bg-indigo-700 text-white hover:shadow-indigo-500/40'}`}
        >
            <input 
                id="resume-upload" 
                type="file" 
                accept=".pdf,.docx" 
                onChange={handleFileChange} 
                className="hidden" 
                disabled={isProcessing}
            />
            {isProcessing ? (
                <>
                    <LoaderIcon />
                    Processing...
                </>
            ) : (
                <>
                    <UploadIcon className="w-5 h-5 mr-2" />
                    {selectedFile ? 'Change File' : 'Select File (PDF/DOCX)'}
                </>
            )}
        </label>
        
        <div className="flex-1 text-sm text-gray-700 p-3 border border-dashed rounded-xl bg-gray-50 flex items-center min-h-[48px]">
            {selectedFile ? (
                <span className="font-semibold text-indigo-700 truncate">{selectedFile.name}</span>
            ) : (
                <span className="text-gray-500">No file chosen.</span>
            )}
        </div>
      </div>
      
      <button
        onClick={startInterview}
        disabled={!selectedFile || isProcessing || interviewPhase !== 'setup'}
        className="mt-2 w-full bg-green-500 hover:bg-green-600 text-white font-bold py-3 px-6 rounded-xl shadow-xl transition duration-300 ease-in-out transform hover:scale-[1.01] disabled:bg-gray-400 disabled:shadow-none disabled:cursor-not-allowed flex items-center justify-center"
      >
        {isProcessing ? (
            'Processing...'
        ) : (
            <>
                <ZapIcon className="w-5 h-5 mr-2" />
                Start AI Interview
            </>
        )}
      </button>
    </div>
  );
    
  const currentQuestion = INTERVIEW_QUESTIONS[currentQuestionIndex];
  const isInputDisabled = isProcessing || interviewPhase === 'setup' || interviewPhase === 'complete' || (interviewPhase === 'running' && !isTimerActive);
  const isSendButtonDisabled = isInputDisabled || !input.trim();


  return (
    <div className="min-h-screen bg-gray-100 font-sans p-4 flex flex-col items-center">
      
      {showWelcomeBackModal && (
          <WelcomeBackModal
              session={session}
              onResume={handleResumeSession}
              onStartNew={handleStartNewSession}
              onClose={handleModalClose}
          />
      )}

      <div className="w-full max-w-4xl bg-white rounded-2xl shadow-2xl shadow-indigo-500/10 overflow-hidden h-[95vh] flex flex-col border border-gray-200">
        
        {/* Header with Branding and Tabs */}
        <header className="p-4 bg-gradient-to-r from-indigo-700 to-indigo-500 text-white shadow-xl">
          <h1 className="text-2xl font-extrabold tracking-wide mb-2">{APP_NAME}</h1>
          <div className="flex space-x-4 border-t border-indigo-400 pt-2">
            
            {/* Interviewee Tab Button */}
            <button
                onClick={() => setActiveTab('interviewee')}
                className={`py-2 px-4 flex items-center font-semibold rounded-t-lg transition duration-200 
                    ${activeTab === 'interviewee' ? 'bg-white text-indigo-700 shadow-t-lg' : 'text-indigo-200 hover:text-white'}`}
            >
                <MessageCircleIcon className="w-5 h-5 mr-2"/> Interviewee Chat
            </button>

            {/* Interviewer Tab Button */}
            <button
                onClick={() => setActiveTab('interviewer')}
                className={`py-2 px-4 flex items-center font-semibold rounded-t-lg transition duration-200 
                    ${activeTab === 'interviewer' ? 'bg-white text-indigo-700 shadow-t-lg' : 'text-indigo-200 hover:text-white'}`}
            >
                <LayoutGridIcon className="w-5 h-5 mr-2"/> Interviewer Dashboard
            </button>
          </div>
        </header>

        {/* --- INTERVIEWEE CHAT CONTENT --- */}
        {activeTab === 'interviewee' && (
            <>
                <main id="chat-main" className="flex-1 overflow-y-auto p-6 space-y-4 bg-gray-100">
                
                {/* Status Bar */}
                <InterviewStatus phase={interviewPhase} />
                
                {/* Timer Display (only shown when interview is running) */}
                {isTimerActive && currentQuestion && (
                    <TimerDisplay 
                        time={timer} 
                        difficulty={currentQuestion.difficulty}
                        totalTime={currentQuestion.timeLimit}
                    />
                )}

                {interviewPhase === 'setup' && <ResumeInputArea />}
                
                {chatHistory.map((message) => (
                    <ChatMessage key={message.id} message={message} />
                ))}
                </main>

                {/* Input Form */}
                <footer className="p-4 border-t border-gray-200 bg-white">
                <form onSubmit={handleSendMessage} className="flex space-x-3">
                    <input
                    type="text"
                    value={session.input}
                    onChange={(e) => setSession(prev => ({ ...prev, input: e.target.value }))}
                    placeholder={
                        interviewPhase === 'setup' ? "Please load resume to begin."
                        : interviewPhase === 'collecting_info' ? `Enter your ${missingField}...`
                        : interviewPhase === 'running' ? `Answer Q${currentQuestionIndex + 1}...`
                        : "Interview is complete."
                    }
                    className="flex-1 p-3 border border-gray-300 rounded-full focus:ring-4 focus:ring-indigo-200 focus:border-indigo-500 transition duration-150 shadow-inner disabled:bg-gray-50 disabled:text-gray-500"
                    disabled={isInputDisabled}
                    />
                    <button
                    type="submit"
                    className="bg-indigo-600 hover:bg-indigo-700 text-white font-semibold p-3 rounded-full shadow-lg shadow-indigo-500/50 transition duration-300 ease-in-out transform hover:scale-105 disabled:bg-gray-400 disabled:shadow-none disabled:cursor-not-allowed"
                    disabled={isSendButtonDisabled}
                    >
                    <SendIcon className="w-6 h-6"/>
                    </button>
                </form>
                </footer>
            </>
        )}

        {/* --- INTERVIEWER DASHBOARD CONTENT --- */}
        {activeTab === 'interviewer' && (
            <main className="flex-1 overflow-y-auto bg-gray-50">
                <InterviewerDashboard 
                    allCandidates={allCandidates} 
                    setCurrentSession={setSession}
                    setActiveTab={setActiveTab}
                />
            </main>
        )}
      </div>
    </div>
  );
}

