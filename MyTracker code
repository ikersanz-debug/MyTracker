import React, { useState, useEffect, createContext, useContext, useMemo } from 'react';
import { initializeApp } from 'firebase/app';
import { getAuth, signInAnonymously, signInWithCustomToken, onAuthStateChanged } from 'firebase/auth';
import { getFirestore, doc, addDoc, setDoc, updateDoc, deleteDoc, onSnapshot, collection, query, where, getDocs, serverTimestamp } from 'firebase/firestore';

// Lucide React Icons
import { BookOpen, Calendar, BarChart2, Timer, Plus, Edit, Trash2, X, CheckCircle, Circle, AlarmCheck, Sun, Moon, ChevronDown, ChevronUp, ChevronLeft, ChevronRight } from 'lucide-react';

// Recharts for interactive charts
import { LineChart, Line, CartesianGrid, XAxis, YAxis, Tooltip, Legend, ResponsiveContainer } from 'recharts';

// Tailwind CSS is assumed to be available

// Global Firebase variables (provided by the Canvas environment)
const appId = typeof __app_id !== 'undefined' ? __app_id : 'default-app-id';
const firebaseConfig = typeof __firebase_config !== 'undefined' ? JSON.parse(__firebase_config) : {};
const initialAuthToken = typeof __initial_auth_token !== 'undefined' ? __initial_auth_token : null; // Corrected variable name

// Context for App State
const AppContext = createContext();

// Custom Hook to use App Context
const useAppContext = () => useContext(AppContext);

const AppProvider = ({ children }) => {
    const [db, setDb] = useState(null);
    const [auth, setAuth] = useState(null);
    const [userId, setUserId] = useState(null);
    const [isAuthReady, setIsAuthReady] = useState(false);
    const [subjects, setSubjects] = useState([]);
    const [studySessions, setStudySessions] = useState([]);
    const [todos, setTodos] = useState([]);
    const [loading, setLoading] = useState(true);
    const [error, setError] = useState(null);

    // State for generic confirmation modal
    const [isConfirmModalOpen, setIsConfirmModalOpen] = useState(false);
    const [confirmModalData, setConfirmModalData] = useState({ title: '', message: '', onConfirm: () => {}, onCancel: () => {} });

    // Pomodoro Timer States (lifted from PomodoroTimer component)
    const [pomodoroMinutes, setPomodoroMinutes] = useState(25);
    const [breakMinutes, setBreakMinutes] = useState(5);
    const [longBreakMinutes, setLongBreakMinutes] = useState(15);
    const [pomodorosBeforeLongBreak, setPomodorosBeforeLongBreak] = useState(4); // New state
    const [minutes, setMinutes] = useState(pomodoroMinutes);
    const [seconds, setSeconds] = useState(0);
    const [isActive, setIsActive] = useState(false);
    const [isBreak, setIsBreak] = useState(false);
    const [sessionsCompleted, setSessionsCompleted] = useState(0);
    const [pomodoroMessage, setPomodoroMessage] = useState('');
    const [pomodoroMessageType, setPomodoroMessageType] = useState('');

    // Helper to open the confirmation modal
    const openConfirmModal = (title, message, onConfirmCallback) => {
        setConfirmModalData({
            title,
            message,
            onConfirm: () => { onConfirmCallback(); setIsConfirmModalOpen(false); },
            onCancel: () => setIsConfirmModalOpen(false)
        });
        setIsConfirmModalOpen(true);
    };

    useEffect(() => {
        // Initialize Firebase and set up auth listener
        const initializeFirebase = async () => {
            try {
                const app = initializeApp(firebaseConfig);
                const firestore = getFirestore(app);
                const authInstance = getAuth(app);
                setDb(firestore);
                setAuth(authInstance);

                // Attempt to sign in with custom token. If it fails, sign in anonymously.
                if (initialAuthToken) {
                    try {
                        await signInWithCustomToken(authInstance, initialAuthToken);
                        console.log("Signed in with custom token successfully.");
                    } catch (tokenError) {
                        console.warn("Failed to sign in with custom token, attempting anonymous sign-in:", tokenError);
                        await signInAnonymously(authInstance);
                    }
                } else {
                    await signInAnonymously(authInstance);
                }

                // Set up auth state listener to get the actual user ID
                const unsubscribe = onAuthStateChanged(authInstance, (user) => {
                    if (user) {
                        setUserId(user.uid);
                    } else {
                        signInAnonymously(authInstance)
                            .then(anonUser => setUserId(anonUser.user.uid))
                            .catch(e => {
                                console.error("Error signing in anonymously after auth state change:", e);
                                setError("Error de autenticación. Por favor, recarga la página.");
                            });
                    }
                    setIsAuthReady(true); // Mark authentication as ready after initial check
                });

                return () => unsubscribe(); // Cleanup auth listener
            } catch (err) {
                console.error("Error initializing Firebase:", err);
                setError("Error al inicializar la aplicación. Por favor, verifica la configuración.");
                setLoading(false);
            }
        };

        initializeFirebase();
    }, []); // Run only once on component mount

    useEffect(() => {
        // Fetch data only when Firebase is initialized and user ID is available
        if (isAuthReady && db && userId) {
            setLoading(true);
            setError(null);

            // Fetch subjects
            const subjectsRef = collection(db, `artifacts/${appId}/users/${userId}/subjects`);
            const unsubscribeSubjects = onSnapshot(subjectsRef, (snapshot) => {
                const fetchedSubjects = snapshot.docs.map(doc => ({ id: doc.id, ...doc.data() }));
                setSubjects(fetchedSubjects);
            }, (err) => {
                console.error("Error fetching subjects:", err);
                setError("Error al cargar las asignaturas.");
            });

            // Fetch study sessions
            const sessionsRef = collection(db, `artifacts/${appId}/users/${userId}/studySessions`);
            const unsubscribeSessions = onSnapshot(sessionsRef, (snapshot) => {
                const fetchedSessions = snapshot.docs.map(doc => ({ id: doc.id, ...doc.data() }));
                setStudySessions(fetchedSessions);
            }, (err) => {
                console.error("Error fetching study sessions:", err);
                setError("Error al cargar las sesiones de estudio.");
            });

            // Fetch todos
            const todosRef = collection(db, `artifacts/${appId}/users/${userId}/todos`);
            const unsubscribeTodos = onSnapshot(todosRef, (snapshot) => {
                const fetchedTodos = snapshot.docs.map(doc => ({ id: doc.id, ...doc.data() }));
                setTodos(fetchedTodos.sort((a,b) => (a.createdAt?.toDate() || 0) - (b.createdAt?.toDate() || 0))); // Sort by creation time
                setLoading(false); // Set loading to false once todos are fetched (last data set)
            }, (err) => {
                console.error("Error fetching todos:", err);
                setError("Error al cargar las tareas pendientes.");
                setLoading(false); // Set loading to false on error too
            });


            return () => {
                unsubscribeSubjects();
                unsubscribeSessions();
                unsubscribeTodos(); // Cleanup todos listener
            }; // Cleanup listeners when component unmounts or dependencies change
        } else if (isAuthReady && !userId) {
            setError("No se pudo obtener el ID de usuario. La autenticación falló.");
            setLoading(false);
        }
    }, [isAuthReady, db, userId]);

    // Pomodoro Timer Logic (lifted from PomodoroTimer component)
    useEffect(() => {
        let interval = null;
        if (isActive) {
            interval = setInterval(() => {
                if (seconds === 0) {
                    if (minutes === 0) {
                        clearInterval(interval);
                        handleTimerEnd();
                    } else {
                        setMinutes(minutes - 1);
                        setSeconds(59);
                    }
                } else {
                    setSeconds(seconds - 1);
                }
            }, 1000);
        } else if (!isActive && seconds !== 0) {
            clearInterval(interval);
        }
        return () => clearInterval(interval);
    }, [isActive, minutes, seconds, isBreak, pomodoroMinutes, breakMinutes, longBreakMinutes, sessionsCompleted, pomodorosBeforeLongBreak]);


    const addSubject = async (subjectData) => {
        if (!db || !userId) {
            console.error("Firestore not initialized or userId not available.");
            return { success: false, error: "App no lista. Inténtalo de nuevo." };
        }
        try {
            await addDoc(collection(db, `artifacts/${appId}/users/${userId}/subjects`), {
                ...subjectData,
                createdAt: serverTimestamp()
            });
            return { success: true };
        } catch (e) {
            console.error("Error adding document: ", e);
            return { success: false, error: e.message };
        }
    };

    const updateSubject = async (id, subjectData) => {
        if (!db || !userId) {
            console.error("Firestore not initialized or userId not available.");
            return { success: false, error: "App no lista. Inténtalo de nuevo." };
        }
        try {
            await setDoc(doc(db, `artifacts/${appId}/users/${userId}/subjects`, id), {
                ...subjectData,
                updatedAt: serverTimestamp()
            }, { merge: true });
            return { success: true };
        } catch (e) {
            console.error("Error updating document: ", e);
            return { success: false, error: e.message };
        }
    };

    const deleteSubject = async (id) => {
        if (!db || !userId) {
            console.error("Firestore not initialized or userId not available.");
            return { success: false, error: "App no lista. Inténtalo de nuevo." };
        }
        try {
            await deleteDoc(doc(db, `artifacts/${appId}/users/${userId}/subjects`, id));
            const sessionsToDeleteQuery = query(collection(db, `artifacts/${appId}/users/${userId}/studySessions`), where("subjectId", "==", id));
            const sessionsSnapshot = await getDocs(sessionsToDeleteQuery);
            sessionsSnapshot.forEach(async (sessionDoc) => {
                await deleteDoc(doc(db, `artifacts/${appId}/users/${userId}/studySessions`, sessionDoc.id));
            });
            return { success: true };
        } catch (e) {
            console.error("Error deleting document: ", e);
            return { success: false, error: e.message };
        }
    };

    const addStudySession = async (sessionData) => {
        if (!db || !userId) {
            console.error("Firestore not initialized or userId not available.");
            return { success: false, error: "App no lista. Inténtalo de nuevo." };
        }
        try {
            await addDoc(collection(db, `artifacts/${appId}/users/${userId}/studySessions`), {
                ...sessionData,
                createdAt: serverTimestamp()
            });
            return { success: true };
        } catch (e) {
            console.error("Error adding session: ", e);
            return { success: false, error: e.message };
        }
    };

    const updateStudySession = async (id, sessionData) => {
        if (!db || !userId) {
            console.error("Firestore not initialized or userId not available.");
            return { success: false, error: "App no lista. Inténtalo de nuevo." };
        }
        try {
            await setDoc(doc(db, `artifacts/${appId}/users/${userId}/studySessions`, id), {
                ...sessionData,
                updatedAt: serverTimestamp()
            }, { merge: true });
            return { success: true };
        } catch (e) {
            console.error("Error updating session: ", e);
            return { success: false, error: e.message };
        }
    };

    const deleteStudySession = async (id) => {
        if (!db || !userId) {
            console.error("Firestore not initialized or userId not available.");
            return { success: false, error: "App no lista. Inténtalo de nuevo." };
        }
        try {
            await deleteDoc(doc(db, `artifacts/${appId}/users/${userId}/studySessions`, id));
            return { success: true };
        } catch (e) {
            console.error("Error deleting session: ", e);
            return { success: false, error: e.message };
        }
    };

    const addTodo = async (todoData) => {
        if (!db || !userId) {
            console.error("Firestore not initialized or userId not available.");
            return { success: false, error: "App no lista. Inténtalo de nuevo." };
        }
        try {
            await addDoc(collection(db, `artifacts/${appId}/users/${userId}/todos`), {
                ...todoData,
                createdAt: serverTimestamp()
            });
            return { success: true };
        } catch (e) {
            console.error("Error adding todo: ", e);
            return { success: false, error: e.message };
        }
    };

    const updateTodo = async (id, todoData) => {
        if (!db || !userId) {
            console.error("Firestore not initialized or userId not available.");
            return { success: false, error: "App no lista. Inténtalo de nuevo." };
        }
        try {
            await setDoc(doc(db, `artifacts/${appId}/users/${userId}/todos`, id), {
                ...todoData,
                updatedAt: serverTimestamp()
            }, { merge: true });
            return { success: true };
        } catch (e) {
            console.error("Error updating todo: ", e);
            return { success: false, error: e.message };
        }
    };

    const deleteTodo = async (id) => {
        if (!db || !userId) {
            console.error("Firestore not initialized or userId not available.");
            return { success: false, error: "App no lista. Inténtalo de nuevo." };
        }
        try {
            await deleteDoc(doc(db, `artifacts/${appId}/users/${userId}/todos`, id));
            return { success: true };
        } catch (e) {
            console.error("Error deleting todo: ", e);
            return { success: false, error: e.message };
        }
    };

    // Pomodoro Timer Control Functions
    const handleStartPausePomodoro = () => {
        // No subject selection needed for Pomodoro
        setPomodoroMessage(''); // Clear previous messages
        setPomodoroMessageType('');
        setIsActive((prev) => !prev);
    };

    const handleResetPomodoro = () => {
        setIsActive(false);
        setMinutes(pomodoroMinutes);
        setSeconds(0);
        setIsBreak(false);
        setSessionsCompleted(0);
        setPomodoroMessage('');
        setPomodoroMessageType('');
    };

    const handleTimerEnd = async () => {
        setIsActive(false);
        if (!isBreak) {
            // Pomodoro session ended
            const sessionDuration = pomodoroMinutes; // The planned duration
            // Add a generic Pomodoro session, no subjectId needed
            const result = await addStudySession({
                subjectId: null, // No specific subject for Pomodoro from here
                date: new Date(),
                duration: sessionDuration,
                type: 'pomodoro',
                description: `Sesión Pomodoro completada (${sessionDuration} min)`
            });
            if (result.success) {
                setPomodoroMessage(`¡Pomodoro completado! ${sessionDuration} minutos registrados.`);
                setPomodoroMessageType('success');
            } else {
                setPomodoroMessage(`Error al registrar Pomodoro: ${result.error}`);
                setPomodoroMessageType('error');
            }

            const newSessionsCompleted = sessionsCompleted + 1;
            setSessionsCompleted(newSessionsCompleted);
            setIsBreak(true);
            if (newSessionsCompleted % pomodorosBeforeLongBreak === 0) { // Use new state
                setMinutes(longBreakMinutes);
            } else {
                setMinutes(breakMinutes);
            }
        } else {
            // Break ended
            setIsBreak(false);
            setMinutes(pomodoroMinutes);
            setPomodoroMessage('¡Descanso terminado! Es hora de otro Pomodoro.');
            setPomodoroMessageType('info');
        }
        setSeconds(0);
    };

    const value = {
        userId,
        subjects,
        studySessions,
        todos,
        loading,
        error,
        addSubject,
        updateSubject,
        deleteSubject,
        addStudySession,
        updateStudySession,
        deleteStudySession,
        addTodo,
        updateTodo,
        deleteTodo,
        openConfirmModal,
        // Pomodoro states and functions
        pomodoroStates: {
            pomodoroMinutes, setPomodoroMinutes,
            breakMinutes, setBreakMinutes,
            longBreakMinutes, setLongBreakMinutes,
            pomodorosBeforeLongBreak, setPomodorosBeforeLongBreak, // New state and setter
            minutes, seconds,
            isActive, isBreak, sessionsCompleted,
            pomodoroMessage, pomodoroMessageType
        },
        pomodoroActions: {
            handleStartPausePomodoro,
            handleResetPomodoro,
            setMinutes, setSeconds // Allow PomodoroTimer to update internal timer display
        }
    };

    if (!isAuthReady) {
        return (
            <div className="flex items-center justify-center min-h-screen bg-gray-100 text-gray-900 dark:bg-gray-900 dark:text-white">
                Cargando autenticación...
            </div>
        );
    }

    if (loading) {
        return (
            <div className="flex items-center justify-center min-h-screen bg-gray-100 text-gray-900 dark:bg-gray-900 dark:text-white">
                Cargando datos...
            </div>
        );
    }

    if (error) {
        return (
            <div className="flex flex-col items-center justify-center min-h-screen bg-gray-100 text-red-600 p-4 text-center dark:bg-gray-900 dark:text-red-400">
                <p className="text-xl mb-4">¡Oops! Ha ocurrido un error:</p>
                <p className="mb-4">{error}</p>
                <p>Por favor, recarga la página o contacta con soporte.</p>
                <p className="text-sm mt-4 text-gray-700 dark:text-gray-500">ID de Usuario: {userId || 'No disponible'}</p>
            </div>
        );
    }

    return (
        <AppContext.Provider value={value}>
            {children}
            {/* Generic Confirmation Modal */}
            <Modal
                title={confirmModalData.title}
                isOpen={isConfirmModalOpen}
                onClose={confirmModalData.onCancel}
            >
                <p className="text-gray-700 mb-4 dark:text-gray-300">{confirmModalData.message}</p>
                <div className="flex justify-end space-x-3">
                    <button
                        onClick={confirmModalData.onCancel}
                        className="px-4 py-2 bg-gray-300 text-gray-800 rounded-md hover:bg-gray-400 transition-colors duration-200 dark:bg-gray-600 dark:text-gray-200 dark:hover:bg-gray-500"
                    >
                        Cancelar
                    </button>
                    <button
                        onClick={confirmModalData.onConfirm}
                        className="px-4 py-2 bg-blue-600 text-white rounded-md hover:bg-blue-700 transition-colors duration-200 dark:bg-blue-700 dark:hover:bg-blue-800"
                    >
                        Confirmar
                    </button>
                </div>
            </Modal>
        </AppContext.Provider>
    );
};


// Reusable Modal Component
const Modal = ({ title, isOpen, onClose, children }) => {
    if (!isOpen) return null;
    return (
        <div className="fixed inset-0 bg-black bg-opacity-75 flex items-center justify-center z-50 p-4">
            <div className="bg-white rounded-lg shadow-xl w-full max-w-md border border-gray-200 dark:bg-gray-800 dark:border-gray-700">
                <div className="flex justify-between items-center p-4 border-b border-gray-200 dark:border-gray-700">
                    <h2 className="text-xl font-semibold text-gray-800 dark:text-white">{title}</h2>
                    <button onClick={onClose} className="text-gray-600 hover:text-gray-900 dark:text-gray-400 dark:hover:text-white">
                        <X size={24} />
                    </button>
                </div>
                <div className="p-4">
                    {children}
                </div>
            </div>
        </div>
    );
};

// Subject Management Component
const SubjectManagement = () => {
    const { subjects, addSubject, updateSubject, deleteSubject, userId, openConfirmModal } = useAppContext();
    const [isAddModalOpen, setIsAddModalOpen] = useState(false);
    const [isEditModalOpen, setIsEditModalOpen] = useState(false);
    const [currentSubject, setCurrentSubject] = useState(null);
    const [subjectName, setSubjectName] = useState('');
    const [professorName, setProfessorName] = useState('');
    const [importantDates, setImportantDates] = useState([]); // [{ type: 'exam', date: 'YYYY-MM-DD', description: '' }]
    const [subjectColor, setSubjectColor] = useState('#6366F1'); // Default color: Indigo 500
    const [message, setMessage] = useState('');
    const [messageType, setMessageType] = useState(''); // 'success' or 'error'

    const resetForm = () => {
        setSubjectName('');
        setProfessorName('');
        setImportantDates([]);
        setCurrentSubject(null);
        setSubjectColor('#6366F1'); // Reset to default color
        setMessage('');
        setMessageType('');
    };

    const handleAddSubject = async () => {
        if (!subjectName.trim()) {
            setMessage('El nombre de la asignatura es obligatorio.');
            setMessageType('error');
            return;
        }
        const result = await addSubject({ name: subjectName, professor: professorName, importantDates, color: subjectColor });
        if (result.success) {
            setMessage('Asignatura añadida con éxito.');
            setMessageType('success');
            setIsAddModalOpen(false);
            resetForm();
        } else {
            setMessage(`Error al añadir asignatura: ${result.error}`);
            setMessageType('error');
        }
    };

    const handleEditSubject = (subject) => {
        setCurrentSubject(subject);
        setSubjectName(subject.name);
        setProfessorName(subject.professor || '');
        setImportantDates(subject.importantDates || []);
        setSubjectColor(subject.color || '#6366F1'); // Set current color or default
        setIsEditModalOpen(true);
    };

    const handleUpdateSubject = async () => {
        if (!currentSubject || !subjectName.trim()) {
            setMessage('El nombre de la asignatura es obligatorio.');
            setMessageType('error');
            return;
        }
        const result = await updateSubject(currentSubject.id, { name: subjectName, professor: professorName, importantDates, color: subjectColor });
        if (result.success) {
            setMessage('Asignatura actualizada con éxito.');
            setMessageType('success');
            setIsEditModalOpen(false);
            resetForm();
        } else {
            setMessage(`Error al actualizar asignatura: ${result.error}`);
            setMessageType('error');
        }
    };

    const handleDeleteSubjectConfirmed = async (id) => {
        const result = await deleteSubject(id);
        if (result.success) {
            setMessage('Asignatura eliminada con éxito.');
            setMessageType('success');
        } else {
            setMessage(`Error al eliminar asignatura: ${result.error}`);
            setMessageType('error');
        }
    };

    const handleDeleteSubject = (id) => {
        openConfirmModal(
            'Confirmar Eliminación',
            '¿Estás seguro de que quieres eliminar esta asignatura? Se eliminarán también todas sus sesiones de estudio asociadas.',
            () => handleDeleteSubjectConfirmed(id)
        );
    };

    const addImportantDate = () => {
        setImportantDates([...importantDates, { type: 'examen', date: '', description: '' }]);
    };

    const updateImportantDate = (index, field, value) => {
        const newDates = [...importantDates];
        newDates[index] = { ...newDates[index], [field]: value };
        setImportantDates(newDates);
    };

    const removeImportantDate = (index) => {
        setImportantDates(importantDates.filter((_, i) => i !== index));
    };

    return (
        <div className="p-4 sm:p-6 lg:p-8 bg-white text-gray-900 min-h-[calc(100vh-64px)] rounded-lg shadow-inner border border-gray-200 dark:bg-gray-900 dark:text-white dark:border-gray-700">
            <h1 className="text-3xl font-bold mb-6 text-center text-blue-700 dark:text-blue-400">Gestión de Asignaturas</h1>
            <p className="text-center text-sm mb-6 text-gray-600 dark:text-gray-400">ID de Usuario: {userId}</p>

            {message && (
                <div className={`p-3 mb-4 rounded-lg text-center text-sm ${messageType === 'success' ? 'bg-green-600 dark:bg-green-700' : 'bg-red-600 dark:bg-red-700'} text-white`}>
                    {message}
                </div>
            )}

            <button
                onClick={() => { setIsAddModalOpen(true); resetForm(); }}
                className="w-full bg-blue-600 hover:bg-blue-700 text-white font-bold py-3 px-4 rounded-lg shadow-md transition duration-300 ease-in-out transform hover:scale-105 flex items-center justify-center mb-6 dark:bg-blue-700 dark:hover:bg-blue-800"
            >
                <Plus size={20} className="mr-2" />
                Añadir Nueva Asignatura
            </button>

            <div className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-6">
                {subjects.length === 0 ? (
                    <p className="text-center text-gray-600 col-span-full dark:text-gray-400">No hay asignaturas añadidas. ¡Añade una para empezar!</p>
                ) : (
                    subjects.map(subject => (
                        <div key={subject.id} className="bg-white p-5 rounded-lg shadow-lg border border-gray-200 dark:bg-gray-800 dark:border-gray-700" style={{ borderColor: subject.color || '#6366F1', borderWidth: '2px' }}>
                            <div>
                                <h3 className="text-xl font-semibold mb-2 dark:text-blue-300" style={{ color: subject.color || '#6366F1' }}>{subject.name}</h3>
                                {subject.professor && <p className="text-gray-700 text-sm mb-3 dark:text-gray-400">Profesor/a: {subject.professor}</p>}
                                {subject.importantDates && subject.importantDates.length > 0 && (
                                    <div className="mb-3">
                                        <h4 className="text-md font-medium text-gray-700 mb-1 dark:text-gray-300">Fechas Importantes:</h4>
                                        <ul className="list-disc list-inside text-gray-700 text-sm dark:text-gray-400">
                                            {subject.importantDates.map((date, idx) => (
                                                <li key={idx}>
                                                    <span className="capitalize">{date.type}</span>: {date.date} - {date.description}
                                                </li>
                                            ))}
                                        </ul>
                                    </div>
                                )}
                            </div>
                            <div className="flex justify-end gap-2 mt-4">
                                <button
                                    onClick={() => handleEditSubject(subject)}
                                    className="bg-blue-600 hover:bg-blue-700 text-white p-2 rounded-full transition duration-300 ease-in-out transform hover:scale-110 dark:bg-blue-700 dark:hover:bg-blue-800" // Changed to blue
                                    title="Editar Asignatura"
                                >
                                    <Edit size={18} />
                                </button>
                                <button
                                    onClick={() => handleDeleteSubject(subject.id)}
                                    className="bg-gray-500 hover:bg-gray-600 text-white p-2 rounded-full transition duration-300 ease-in-out transform hover:scale-110 dark:bg-gray-700 dark:hover:bg-gray-800" // Changed to gray
                                    title="Eliminar Asignatura"
                                >
                                    <Trash2 size={18} />
                                </button>
                            </div>
                        </div>
                    ))
                )}
            </div>

            {/* Add Subject Modal */}
            <Modal title="Añadir Nueva Asignatura" isOpen={isAddModalOpen} onClose={() => setIsAddModalOpen(false)}>
                <div className="flex flex-col space-y-4">
                    {message && (
                        <div className={`p-2 rounded text-sm ${messageType === 'success' ? 'bg-green-600 dark:bg-green-700' : 'bg-red-600 dark:bg-red-700'} text-white`}>
                            {message}
                        </div>
                    )}
                    <input
                        type="text"
                        placeholder="Nombre de la Asignatura"
                        value={subjectName}
                        onChange={(e) => setSubjectName(e.target.value)}
                        className="p-3 bg-gray-50 text-gray-900 rounded-md border border-gray-300 focus:ring-blue-500 focus:border-blue-500 dark:bg-gray-700 dark:text-white dark:border-gray-600"
                    />
                    <input
                        type="text"
                        placeholder="Nombre del Profesor/a (Opcional)"
                        value={professorName}
                        onChange={(e) => setProfessorName(e.target.value)}
                        className="p-3 bg-gray-50 text-gray-900 rounded-md border border-gray-300 focus:ring-blue-500 focus:border-blue-500 dark:bg-gray-700 dark:text-white dark:border-gray-600"
                    />
                    <div className="flex items-center space-x-2">
                        <label htmlFor="subject-color" className="text-gray-700 dark:text-gray-300">Color:</label>
                        <input
                            type="color"
                            id="subject-color"
                            value={subjectColor}
                            onChange={(e) => setSubjectColor(e.target.value)}
                            className="w-10 h-10 p-1 border border-gray-300 rounded-md cursor-pointer dark:border-gray-600"
                            title="Selecciona un color para la asignatura"
                        />
                        <span className="text-sm text-gray-600 dark:text-gray-400">{subjectColor}</span>
                    </div>

                    <h3 className="text-gray-700 font-medium mt-2 dark:text-gray-300">Fechas Importantes:</h3>
                    {importantDates.map((date, index) => (
                        <div key={index} className="flex flex-col sm:flex-row gap-2 items-center bg-gray-100 p-3 rounded-md dark:bg-gray-700">
                            <select
                                value={date.type}
                                onChange={(e) => updateImportantDate(index, 'type', e.target.value)}
                                className="flex-1 p-2 bg-gray-200 text-gray-900 rounded-md border border-gray-300 dark:bg-gray-600 dark:text-white dark:border-gray-500"
                            >
                                <option value="examen">Examen</option>
                                <option value="entrega">Entrega</option>
                                <option value="clase">Clase</option>
                                <option value="otro">Otro</option>
                            </select>
                            <input
                                type="date"
                                value={date.date}
                                onChange={(e) => updateImportantDate(index, 'date', e.target.value)}
                                className="flex-1 p-2 bg-gray-200 text-gray-900 rounded-md border border-gray-300 dark:bg-gray-600 dark:text-white dark:border-gray-500"
                            />
                            <input
                                type="text"
                                placeholder="Descripción"
                                value={date.description}
                                onChange={(e) => updateImportantDate(index, 'description', e.target.value)}
                                className="flex-1 p-2 bg-gray-200 text-gray-900 rounded-md border border-gray-300 dark:bg-gray-600 dark:text-white dark:border-gray-500 min-w-0" // Added min-w-0
                            />
                            <button
                                onClick={() => removeImportantDate(index)}
                                className="bg-gray-500 hover:bg-gray-600 text-white p-2 rounded-md dark:bg-gray-700 dark:hover:bg-gray-800" // Changed to gray
                            >
                                <Trash2 size={18} />
                            </button>
                        </div>
                    ))}
                    <button
                        onClick={addImportantDate}
                        className="bg-gray-200 hover:bg-gray-300 text-gray-800 py-2 px-4 rounded-md border border-gray-300 flex items-center justify-center text-sm dark:bg-gray-700 dark:hover:bg-gray-600 dark:border-gray-600"
                    >
                        <Plus size={16} className="mr-1" /> Añadir Fecha
                    </button>
                    <button
                        onClick={handleAddSubject}
                        className="bg-blue-600 hover:bg-blue-700 text-white font-bold py-3 px-4 rounded-md mt-4 dark:bg-blue-700 dark:hover:bg-blue-800"
                    >
                        Guardar Asignatura
                    </button>
                </div>
            </Modal>

            {/* Edit Subject Modal */}
            <Modal title="Editar Asignatura" isOpen={isEditModalOpen} onClose={() => setIsEditModalOpen(false)}>
                <div className="flex flex-col space-y-4">
                    {message && (
                        <div className={`p-2 rounded text-sm ${messageType === 'success' ? 'bg-green-600 dark:bg-green-700' : 'bg-red-600 dark:bg-red-700'} text-white`}>
                            {message}
                        </div>
                    )}
                    <input
                        type="text"
                        placeholder="Nombre de la Asignatura"
                        value={subjectName}
                        onChange={(e) => setSubjectName(e.target.value)}
                        className="p-3 bg-gray-50 text-gray-900 rounded-md border border-gray-300 focus:ring-blue-500 focus:border-blue-500 dark:bg-gray-700 dark:text-white dark:border-gray-600"
                    />
                    <input
                        type="text"
                        placeholder="Nombre del Profesor/a"
                        value={professorName}
                        onChange={(e) => setProfessorName(e.target.value)}
                        className="p-3 bg-gray-50 text-gray-900 rounded-md border border-gray-300 focus:ring-blue-500 focus:border-blue-500 dark:bg-gray-700 dark:text-white dark:border-gray-600"
                    />
                    <div className="flex items-center space-x-2">
                        <label htmlFor="subject-color-edit" className="text-gray-700 dark:text-gray-300">Color:</label>
                        <input
                            type="color"
                            id="subject-color-edit"
                            value={subjectColor}
                            onChange={(e) => setSubjectColor(e.target.value)}
                            className="w-10 h-10 p-1 border border-gray-300 rounded-md cursor-pointer dark:border-gray-600"
                            title="Selecciona un color para la asignatura"
                        />
                        <span className="text-sm text-gray-600 dark:text-gray-400">{subjectColor}</span>
                    </div>
                    <h3 className="text-gray-700 font-medium mt-2 dark:text-gray-300">Fechas Importantes:</h3>
                    {importantDates.map((date, index) => (
                        <div key={index} className="flex flex-col sm:flex-row gap-2 items-center bg-gray-100 p-3 rounded-md dark:bg-gray-700">
                            <select
                                value={date.type}
                                onChange={(e) => updateImportantDate(index, 'type', e.target.value)}
                                className="flex-1 p-2 bg-gray-200 text-gray-900 rounded-md border border-gray-300 dark:bg-gray-600 dark:text-white dark:border-gray-500"
                            >
                                <option value="examen">Examen</option>
                                <option value="entrega">Entrega</option>
                                <option value="clase">Clase</option>
                                <option value="otro">Otro</option>
                            </select>
                            <input
                                type="date"
                                value={date.date}
                                onChange={(e) => updateImportantDate(index, 'date', e.target.value)}
                                className="flex-1 p-2 bg-gray-200 text-gray-900 rounded-md border border-gray-300 dark:bg-gray-600 dark:text-white dark:border-gray-500"
                            />
                            <input
                                type="text"
                                placeholder="Descripción"
                                value={date.description}
                                onChange={(e) => updateImportantDate(index, 'description', e.target.value)}
                                className="flex-1 p-2 bg-gray-200 text-gray-900 rounded-md border border-gray-300 dark:bg-gray-600 dark:text-white dark:border-gray-500 min-w-0" // Added min-w-0
                            />
                            <button
                                onClick={() => removeImportantDate(index)}
                                className="bg-gray-500 hover:bg-gray-600 text-white p-2 rounded-md dark:bg-gray-700 dark:hover:bg-gray-800" // Changed to gray
                            >
                                <Trash2 size={18} />
                            </button>
                        </div>
                    ))}
                    <button
                        onClick={addImportantDate}
                        className="bg-gray-200 hover:bg-gray-300 text-gray-800 py-2 px-4 rounded-md border border-gray-300 flex items-center justify-center text-sm dark:bg-gray-700 dark:hover:bg-gray-600 dark:border-gray-600"
                    >
                        <Plus size={16} className="mr-1" /> Añadir Fecha
                    </button>
                    <button
                        onClick={handleUpdateSubject}
                        className="bg-blue-600 hover:bg-blue-700 text-white font-bold py-3 px-4 rounded-md mt-4 dark:bg-blue-700 dark:hover:bg-blue-800"
                    >
                        Actualizar Asignatura
                    </button>
                </div>
            </Modal>
        </div>
    );
};

// To-do List Component
const TodoList = () => {
    const { todos, addTodo, updateTodo, deleteTodo, userId, openConfirmModal } = useAppContext();
    const [newTodoText, setNewTodoText] = useState('');
    const [message, setMessage] = useState('');
    const [messageType, setMessageType] = useState(''); // 'success' or 'error'

    const handleAddTodo = async () => {
        if (!newTodoText.trim()) {
            setMessage('La descripción de la tarea no puede estar vacía.');
            setMessageType('error');
            return;
        }
        const result = await addTodo({ text: newTodoText, completed: false });
        if (result.success) {
            setNewTodoText('');
        } else {
            setMessage(`Error al añadir tarea: ${result.error}`);
            setMessageType('error');
        }
    };

    const handleToggleComplete = async (todo) => {
        const result = await updateTodo(todo.id, { completed: !todo.completed });
        if (!result.success) { // Only show error message
            setMessage(`Error al actualizar tarea: ${result.error}`);
            setMessageType('error');
        }
    };

    const handleDeleteTodoConfirmed = async (id) => {
        const result = await deleteTodo(id);
        if (!result.success) { // Only show error message
            setMessage(`Error al eliminar tarea: ${result.error}`);
            setMessageType('error');
        }
    };

    const handleDeleteTodo = (id) => {
        openConfirmModal(
            'Confirmar Eliminación',
            '¿Estás seguro de que quieres eliminar esta tarea?',
            () => handleDeleteTodoConfirmed(id)
        );
    };

    const handleClearCompletedConfirmed = async () => {
        const completedTodos = todos.filter(todo => todo.completed);
        let errorOccurred = false;
        for (const todo of completedTodos) {
            const result = await deleteTodo(todo.id);
            if (!result.success) {
                errorOccurred = true;
                setMessage(`Error al eliminar algunas tareas: ${result.error}`);
                setMessageType('error');
                break; // Stop on first error, or accumulate errors for a more detailed message
            }
        }
    };

    const handleClearCompleted = () => {
        openConfirmModal(
            'Limpiar Tareas Completadas',
            '¿Estás seguro de que quieres eliminar todas las tareas completadas?',
            handleClearCompletedConfirmed
        );
    };

    return (
        <div className="p-4 sm:p-6 lg:p-8 bg-white text-gray-900 min-h-[calc(100vh-64px)] rounded-lg shadow-inner border border-gray-200 dark:bg-gray-900 dark:text-white dark:border-gray-700">
            <h1 className="text-3xl font-bold mb-6 text-center text-blue-700 dark:text-blue-400">Lista de Tareas Pendientes</h1>
            <p className="text-center text-sm mb-6 text-gray-600 dark:text-gray-400">ID de Usuario: {userId}</p>

            {message && (
                <div className={`p-3 mb-4 rounded-lg text-center text-sm ${messageType === 'success' ? 'bg-green-600 dark:bg-green-700' : 'bg-red-600 dark:bg-red-700'} text-white`}>
                    {message}
                </div>
            )}

            <div className="flex flex-col sm:flex-row gap-3 mb-6">
                <input
                    type="text"
                    placeholder="Añadir nueva tarea..."
                    value={newTodoText}
                    onChange={(e) => setNewTodoText(e.target.value)}
                    onKeyPress={(e) => e.key === 'Enter' && handleAddTodo()}
                    className="flex-grow p-3 bg-gray-50 text-gray-900 rounded-md border border-gray-300 focus:ring-blue-500 focus:border-blue-500 dark:bg-gray-700 dark:text-white dark:border-gray-600"
                />
                <button
                    onClick={handleAddTodo}
                    className="bg-blue-600 hover:bg-blue-700 text-white font-bold py-3 px-4 rounded-lg shadow-md transition duration-300 ease-in-out transform hover:scale-105 flex items-center justify-center dark:bg-blue-700 dark:hover:bg-blue-800"
                >
                    <Plus size={20} className="mr-2 sm:mr-0 sm:hidden lg:mr-2 lg:inline" />
                    Añadir Tarea
                </button>
            </div>

            {todos.length === 0 ? (
                <p className="text-center text-gray-600 dark:text-gray-400">No hay tareas pendientes. ¡Añade una para empezar!</p>
            ) : (
                <ul className="space-y-3">
                    {todos.map(todo => (
                        <li key={todo.id} className="bg-white p-4 rounded-lg shadow-sm border border-gray-200 flex items-center justify-between dark:bg-gray-800 dark:border-gray-700">
                            <div className="flex items-center">
                                <button
                                    onClick={() => handleToggleComplete(todo)}
                                    className="mr-3 text-gray-500 hover:text-blue-600 dark:text-gray-400 dark:hover:text-blue-400 transition-colors duration-200"
                                    title={todo.completed ? 'Marcar como incompleta' : 'Marcar como completa'}
                                >
                                    {todo.completed ? <CheckCircle size={24} className="text-blue-600" /> : <Circle size={24} />} {/* Changed to blue */}
                                </button>
                                <span className={`text-lg ${todo.completed ? 'line-through text-gray-500 dark:text-gray-400' : 'text-gray-900 dark:text-white'}`}>
                                    {todo.text}
                                </span>
                            </div>
                            <button
                                onClick={() => handleDeleteTodo(todo.id)}
                                className="text-gray-500 hover:text-gray-700 p-2 rounded-full transition duration-300 ease-in-out transform hover:scale-110 dark:text-gray-400 dark:hover:text-gray-200" // Changed to gray
                                title="Eliminar tarea"
                            >
                                <Trash2 size={20} />
                            </button>
                        </li>
                    ))}
                </ul>
            )}

            {todos.filter(todo => todo.completed).length > 0 && (
                <button
                    onClick={handleClearCompleted}
                    className="w-full bg-gray-500 hover:bg-gray-600 text-white font-bold py-3 px-4 rounded-lg shadow-md transition duration-300 ease-in-out transform hover:scale-105 flex items-center justify-center mt-6 dark:bg-gray-700 dark:hover:bg-gray-800" // Changed to gray
                >
                    <Trash2 size={20} className="mr-2" />
                    Limpiar tareas completadas
                </button>
            )}
        </div>
    );
};


// Calendar and Study Planning Component
const CalendarView = () => {
    const { subjects, studySessions, addStudySession, updateStudySession, deleteStudySession, userId, openConfirmModal } = useAppContext();
    const [currentDate, setCurrentDate] = useState(new Date()); // State for the currently displayed date
    const [view, setView] = useState('monthly'); // 'daily', 'weekly', 'monthly'
    const [isAddSessionModalOpen, setIsAddSessionModalOpen] = useState(false);
    const [isEditSessionModalOpen, setIsEditSessionModal] = useState(false);
    const [currentSession, setCurrentSession] = useState(null);
    const [sessionSubjectId, setSessionSubjectId] = useState('');
    const [sessionDate, setSessionDate] = useState('');
    const [sessionStartTime, setSessionStartTime] = useState('');
    const [sessionEndTime, setSessionEndTime] = useState('');
    const [sessionDuration, setSessionDuration] = useState(''); // in minutes
    const [sessionType, setSessionType] = useState('study'); // 'study', 'exam', 'assignment', 'class'
    const [sessionDescription, setSessionDescription] = useState('');
    const [message, setMessage] = useState('');
    const [messageType, setMessageType] = useState('');

    const resetSessionForm = () => {
        setSessionSubjectId('');
        setSessionDate('');
        setSessionStartTime('');
        setSessionEndTime('');
        setSessionDuration('');
        setSessionType('study');
        setSessionDescription('');
        setCurrentSession(null);
        setMessage('');
        setMessageType('');
    };

    const getMonthName = (date) => {
        return date.toLocaleString('es-ES', { month: 'long', year: 'numeric' });
    };

    const getDayName = (date) => {
        return date.toLocaleString('es-ES', { weekday: 'long' });
    };

    const normalizeDate = (date) => {
        const d = new Date(date);
        d.setHours(0, 0, 0, 0);
        return d;
    };

    // Helper to format Date object to 'YYYY-MM-DD' string in local timezone
    const formatDateToYYYYMMDD = (date) => {
        const year = date.getFullYear();
        const month = String(date.getMonth() + 1).padStart(2, '0');
        const day = String(date.getDate()).padStart(2, '0');
        return `${year}-${month}-${day}`;
    };

    const getStartOfWeek = (date) => {
        const d = normalizeDate(date);
        const day = d.getDay(); // Sunday - Saturday : 0 - 6
        const diff = d.getDate() - day + (day === 0 ? -6 : 1); // Adjust to start on Monday
        return normalizeDate(d.setDate(diff));
    };

    const getDaysInMonth = (date) => {
        const year = date.getFullYear();
        const month = date.getMonth();
        return new Date(year, month + 1, 0).getDate();
    };

    const getFirstDayOfMonth = (date) => {
        const year = date.getFullYear();
        const month = date.getMonth();
        return new Date(year, month, 1).getDay(); // 0 for Sunday, 1 for Monday etc.
    };

    const handlePrev = () => {
        if (view === 'monthly') {
            setCurrentDate(new Date(currentDate.getFullYear(), currentDate.getMonth() - 1, 1));
        } else if (view === 'weekly') {
            const newDate = new Date(currentDate);
            newDate.setDate(newDate.getDate() - 7);
            setCurrentDate(newDate);
        } else { // daily
            const newDate = new Date(currentDate);
            newDate.setDate(newDate.getDate() - 1);
            setCurrentDate(newDate);
        }
    };

    const handleNext = () => {
        if (view === 'monthly') {
            setCurrentDate(new Date(currentDate.getFullYear(), currentDate.getMonth() + 1, 1));
        } else if (view === 'weekly') {
            const newDate = new Date(currentDate);
            newDate.setDate(newDate.getDate() + 7);
            setCurrentDate(newDate);
        } else { // daily
            const newDate = new Date(currentDate);
            newDate.setDate(newDate.getDate() + 1);
            setCurrentDate(newDate);
        }
    };

    // Memoize filtered sessions and events for performance and consistency
    const getFilteredData = useMemo(() => {
        return studySessions.map(session => ({
            ...session,
            normalizedDate: normalizeDate(session.date instanceof Date ? session.date : (session.date && session.date.toDate ? session.date.toDate() : new Date(0)))
        })).concat(
            subjects.flatMap(sub =>
                (sub.importantDates || []).map(d => ({
                    id: `sub-${sub.id}-${d.date}-${d.type}-${d.description}`, // Unique ID for events from subjects
                    subjectId: sub.id, // Link event to subject
                    type: d.type,
                    description: d.description,
                    date: new Date(d.date + 'T00:00:00'), // Ensure event date is a Date object
                    normalizedDate: normalizeDate(new Date(d.date + 'T00:00:00')),
                    subjectName: sub.name
                }))
            )
        );
    }, [studySessions, subjects]);


    const renderMonthlyView = () => {
        const year = currentDate.getFullYear();
        const month = currentDate.getMonth();
        const daysInMonth = getDaysInMonth(currentDate);
        const firstDay = getFirstDayOfMonth(currentDate); // 0 (Sunday) to 6 (Saturday)

        const calendarDays = [];
        const today = normalizeDate(new Date());

        // Adjust firstDay to start from Monday (0 for Monday, 6 for Sunday)
        const adjustedFirstDay = (firstDay === 0) ? 6 : firstDay - 1;

        // Add empty cells for days before the first of the month
        for (let i = 0; i < adjustedFirstDay; i++) {
            calendarDays.push(<div key={`empty-${i}`} className="p-2 border border-gray-200 bg-gray-50 text-gray-400 min-h-[100px] dark:border-gray-700 dark:bg-gray-800"></div>);
        }

        for (let day = 1; day <= daysInMonth; day++) {
            const currentDayDate = normalizeDate(new Date(year, month, day));
            const isToday = currentDayDate.getTime() === today.getTime();

            const dayActivities = getFilteredData.filter(activity =>
                activity.normalizedDate.getTime() === currentDayDate.getTime()
            );

            const sessionsForDisplay = dayActivities.filter(a => a.type === 'study' || a.type === 'pomodoro');
            const eventsForDisplay = dayActivities.filter(a => a.type !== 'study' && a.type !== 'pomodoro');

            calendarDays.push(
                <div
                    key={day}
                    className={`p-2 border border-gray-200 min-h-[100px] flex flex-col ${isToday ? 'bg-blue-100 border-blue-400 dark:bg-blue-900 dark:border-blue-500' : 'bg-white dark:bg-gray-800'}`}
                >
                    <span className={`font-bold ${isToday ? 'text-blue-600 dark:text-blue-200' : 'text-gray-800 dark:text-white'}`}>{day}</span>
                    <div className="flex flex-col gap-1 mt-1 text-sm overflow-hidden">
                        {sessionsForDisplay.map(session => {
                            const subject = subjects.find(s => s.id === session.subjectId);
                            const subjectName = subject?.name || 'Sin Asignatura';
                            const itemColor = subject?.color || '#34D399'; // Default green for study
                            return (
                                <div key={session.id} className="text-white rounded-md px-2 py-1 truncate text-xs flex items-center justify-between" style={{ backgroundColor: itemColor }}>
                                    <span className="truncate">{session.description || `${subjectName} (${session.duration} min)`}</span>
                                    {session.type !== 'pomodoro' && ( // Allow editing for non-pomodoro sessions added manually
                                        <button onClick={() => handleEditSession(session)} className="ml-1 text-white/90 hover:text-white dark:text-white/70 dark:hover:text-white"><Edit size={12} /></button>
                                    )}
                                </div>
                            );
                        })}
                        {eventsForDisplay.map((event, idx) => {
                            const subject = subjects.find(s => s.id === event.subjectId);
                            const itemColor = subject?.color || '#8B5CF6'; // Default purple for events
                            return (
                                <div key={`event-${event.id || idx}`} className="text-white rounded-md px-2 py-1 truncate text-xs flex items-center" style={{ backgroundColor: itemColor }}>
                                    <AlarmCheck size={12} className="mr-1" />
                                    <span className="truncate">{event.description || `${event.type} (${event.subjectName})`}</span>
                                    {event.id && event.id.startsWith('sub-') && ( // Only allow editing for subject-related events if they have an ID
                                         <button onClick={() => handleEditSession(event)} className="ml-1 text-white/90 hover:text-white dark:text-white/70 dark:hover:text-white"><Edit size={12} /></button>
                                    )}
                                </div>
                            );
                        })}
                    </div>
                    <button
                        onClick={() => {
                            resetSessionForm();
                            setSessionDate(formatDateToYYYYMMDD(currentDayDate)); // Use formatted date
                            setIsAddSessionModalOpen(true);
                        }}
                        className="mt-auto self-end text-blue-600 hover:text-blue-700 text-sm dark:text-blue-400 dark:hover:text-blue-300"
                    >
                        <Plus size={16} />
                    </button>
                </div>
            );
        }

        return (
            <div className="grid grid-cols-7 gap-px">
                {['Lun', 'Mar', 'Mié', 'Jue', 'Vie', 'Sáb', 'Dom'].map(day => (
                    <div key={day} className="text-center font-semibold text-blue-700 py-2 bg-gray-100 border border-gray-200 dark:bg-gray-800 dark:border-gray-700 dark:text-blue-300">
                        {day}
                    </div>
                ))}
                {calendarDays}
            </div>
        );
    };

    const renderWeeklyView = () => {
        const startOfWeek = getStartOfWeek(currentDate);
        const weekDays = Array.from({ length: 7 }).map((_, i) => {
            const day = new Date(startOfWeek);
            day.setDate(startOfWeek.getDate() + i);
            return day;
        });

        return (
            <div className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-6">
                {weekDays.map(day => {
                    const currentDayDate = normalizeDate(day);
                    const isToday = currentDayDate.getTime() === normalizeDate(new Date()).getTime();

                    const dayActivities = getFilteredData.filter(activity =>
                        activity.normalizedDate.getTime() === currentDayDate.getTime()
                    );

                    const sessionsForDisplay = dayActivities.filter(a => a.type === 'study' || a.type === 'pomodoro');
                    const eventsForDisplay = dayActivities.filter(a => a.type !== 'study' && a.type !== 'pomodoro');


                    return (
                        <div key={day.toDateString()} className={`bg-white p-4 rounded-lg shadow-lg border border-gray-200 dark:bg-gray-800 dark:border-gray-700 ${isToday ? 'border-blue-400 bg-blue-100/50 dark:border-blue-500 dark:bg-blue-900/50' : ''}`}>
                            <h3 className={`text-xl font-semibold mb-3 ${isToday ? 'text-blue-700 dark:text-blue-300' : 'text-gray-900 dark:text-white'}`}>
                                {getDayName(day)}, {day.getDate()} de {day.toLocaleString('es-ES', { month: 'long' })}
                            </h3>
                            <div className="space-y-3">
                                {dayActivities.length === 0 && (
                                    <p className="text-gray-600 dark:text-gray-400">No hay eventos para este día.</p>
                                )}
                                {sessionsForDisplay.map(session => {
                                    const subject = subjects.find(s => s.id === session.subjectId);
                                    const subjectName = subject?.name || 'Sin Asignatura';
                                    const itemColor = subject?.color || '#34D399';
                                    return (
                                        <div key={session.id} className="p-3 rounded-md text-white flex items-center justify-between shadow-sm" style={{ backgroundColor: itemColor }}>
                                            <div>
                                                <p className="font-semibold text-lg">{session.description || subjectName}</p>
                                                <p className="text-sm text-opacity-80 dark:text-opacity-70">
                                                    {session.type === 'study' ? `Estudio: ${session.duration} min` : session.type.charAt(0).toUpperCase() + session.type.slice(1)}
                                                    {session.startTime && session.endTime && ` (${session.startTime} - ${session.endTime})`}
                                                </p>
                                            </div>
                                            {session.type !== 'pomodoro' && (
                                                <button onClick={() => handleEditSession(session)} className="text-white/90 hover:text-white dark:text-white/70 dark:hover:text-white">
                                                    <Edit size={18} />
                                                </button>
                                            )}
                                        </div>
                                    );
                                })}
                                {eventsForDisplay.map((event, idx) => {
                                    const subject = subjects.find(s => s.id === event.subjectId);
                                    const itemColor = subject?.color || '#8B5CF6';
                                    return (
                                        <div key={`event-weekly-${event.id || idx}`} className="p-3 rounded-md text-white flex items-center justify-between shadow-sm" style={{ backgroundColor: itemColor }}>
                                            <div>
                                                <p className="font-semibold text-lg">{event.description}</p>
                                                <p className="text-sm text-opacity-80 dark:text-opacity-70">
                                                    {event.type.charAt(0).toUpperCase() + event.type.slice(1)} de {event.subjectName}
                                                </p>
                                            </div>
                                            {event.id && event.id.startsWith('sub-') && (
                                                <button onClick={() => handleEditSession(event)} className="ml-1 text-white/90 hover:text-white dark:text-white/70 dark:hover:text-white"><Edit size={12} /></button>
                                            )}
                                        </div>
                                    );
                                })}
                            </div>
                            <button
                                onClick={() => {
                                    resetSessionForm();
                                    setSessionDate(formatDateToYYYYMMDD(currentDayDate)); // Use formatted date
                                    setIsAddSessionModalOpen(true);
                                }}
                                className="mt-4 w-full bg-blue-600 hover:bg-blue-700 text-white py-2 rounded-md flex items-center justify-center text-sm dark:bg-blue-700 dark:hover:bg-blue-800"
                            >
                                <Plus size={16} className="mr-1" /> Añadir Sesión/Evento
                            </button>
                        </div>
                    );
                })}
            </div>
        );
    };

    const renderDailyView = () => {
        const currentDayDate = normalizeDate(currentDate);
        const isToday = currentDayDate.getTime() === normalizeDate(new Date()).getTime();

        const dayActivities = getFilteredData.filter(activity =>
            activity.normalizedDate.getTime() === currentDayDate.getTime()
        );
        const sessionsForDisplay = dayActivities.filter(a => a.type === 'study' || a.type === 'pomodoro');
        const eventsForDisplay = dayActivities.filter(a => a.type !== 'study' && a.type !== 'pomodoro');


        return (
            <div className="bg-white p-6 rounded-lg shadow-lg border border-gray-200 max-w-2xl mx-auto dark:bg-gray-800 dark:border-gray-700">
                <h3 className={`text-2xl font-semibold mb-4 ${isToday ? 'text-blue-700 dark:text-blue-300' : 'text-gray-900 dark:text-white'}`}>
                    {getDayName(currentDate)}, {currentDate.getDate()} de {currentDate.toLocaleString('es-ES', { month: 'long' })} de {currentDate.getFullYear()}
                </h3>
                <div className="space-y-4">
                    {dayActivities.length === 0 ? (
                        <p className="text-gray-600 text-center py-4 dark:text-gray-400">No hay sesiones ni eventos planificados para este día.</p>
                    ) : (
                        <>
                            {sessionsForDisplay.map(session => {
                                const subject = subjects.find(s => s.id === session.subjectId);
                                const subjectName = subject?.name || 'Sin Asignatura';
                                const itemColor = subject?.color || '#34D399';
                                return (
                                    <div key={session.id} className="p-4 rounded-md text-white flex items-center justify-between shadow-md" style={{ backgroundColor: itemColor }}>
                                        <div>
                                            <p className="font-bold text-xl">{session.description || subjectName}</p>
                                            <p className="text-sm text-opacity-80 dark:text-opacity-70">
                                                {session.type === 'study' ? `Estudio: ${session.duration} min` : session.type.charAt(0).toUpperCase() + session.type.slice(1)}
                                                {session.startTime && session.endTime && ` (${session.startTime} - ${session.endTime})`}
                                            </p>
                                        </div>
                                        <div className="flex gap-2">
                                            {session.type !== 'pomodoro' && (
                                                <button onClick={() => handleEditSession(session)} className="text-white/90 hover:text-white dark:text-white/80 dark:hover:text-white">
                                                    <Edit size={20} />
                                                </button>
                                            )}
                                            <button onClick={() => handleDeleteSession(session.id)} className="text-gray-500 hover:text-gray-700 dark:text-gray-400 dark:hover:text-gray-200"> {/* Changed to gray */}
                                                <Trash2 size={20} />
                                            </button>
                                        </div>
                                    </div>
                                );
                            })}
                            {eventsForDisplay.map((event, idx) => {
                                const subject = subjects.find(s => s.id === event.subjectId);
                                const itemColor = subject?.color || '#8B5CF6';
                                return (
                                    <div key={`event-daily-${event.id || idx}`} className="p-4 rounded-md text-white flex items-center justify-between shadow-md" style={{ backgroundColor: itemColor }}>
                                        <div>
                                            <p className="font-bold text-xl">{event.description}</p>
                                            <p className="text-sm text-opacity-80 dark:text-opacity-70">
                                                {event.type.charAt(0).toUpperCase() + event.type.slice(1)} de {event.subjectName}
                                            </p>
                                        </div>
                                        {event.id && event.id.startsWith('sub-') && (
                                            <button onClick={() => handleEditSession(event)} className="ml-1 text-white/90 hover:text-white dark:text-white/70 dark:hover:text-white"><Edit size={12} /></button>
                                        )}
                                    </div>
                                );
                            })}
                        </>
                    )}
                </div>
                <button
                    onClick={() => {
                        resetSessionForm();
                        setSessionDate(formatDateToYYYYMMDD(currentDayDate)); // Use formatted date
                        setIsAddSessionModalOpen(true);
                    }}
                    className="mt-6 w-full bg-blue-600 hover:bg-blue-700 text-white py-3 rounded-md flex items-center justify-center font-bold dark:bg-blue-700 dark:hover:bg-blue-800"
                >
                    <Plus size={20} className="mr-2" /> Añadir Sesión/Evento para Hoy
                </button>
            </div>
        );
    };


    const handleAddSession = async () => {
        if (!sessionSubjectId && sessionType === 'study') {
            setMessage('Debe seleccionar una asignatura para la sesión de estudio.');
            setMessageType('error');
            return;
        }
        if (!sessionDate) {
            setMessage('Debe seleccionar una fecha para la sesión.');
            setMessageType('error');
            return;
        }
        if (sessionType === 'study' && !sessionDuration && (!sessionStartTime || !sessionEndTime)) {
            setMessage('Debe especificar la duración o las horas de inicio/fin para la sesión de estudio.');
            setMessageType('error');
            return;
        }
        if (sessionType !== 'study' && !sessionDescription.trim()) {
            setMessage('Debe añadir una descripción para el evento académico.');
            setMessageType('error');
            return;
        }

        let calculatedDuration = parseInt(sessionDuration, 10) || 0;
        let sessionDateTime = normalizeDate(new Date(sessionDate));

        if (sessionStartTime && sessionEndTime) {
            const startMs = new Date(`${sessionDate}T${sessionStartTime}:00`).getTime();
            let endMs = new Date(`${sessionDate}T${sessionEndTime}:00`).getTime();

            // Handle overnight sessions: if end time is earlier than start time, assume it's next day
            if (endMs < startMs) {
                const nextDayEndDt = new Date(`${sessionDate}T${sessionEndTime}:00`);
                nextDayEndDt.setDate(nextDayEndDt.getDate() + 1);
                endMs = nextDayEndDt.getTime();
            }
            calculatedDuration = Math.round((endMs - startMs) / (1000 * 60));
        }


        const sessionData = {
            subjectId: sessionSubjectId || null,
            date: sessionDateTime,
            duration: calculatedDuration,
            type: sessionType,
            description: sessionDescription.trim(),
            startTime: sessionStartTime || null,
            endTime: sessionEndTime || null
        };

        const result = await addStudySession(sessionData);
        if (result.success) {
            setMessage('Sesión/Evento añadido con éxito.');
            setMessageType('success');
            setIsAddSessionModalOpen(false);
            resetSessionForm();
        } else {
            setMessage(`Error al añadir sesión/evento: ${result.error}`);
            setMessageType('error');
        }
    };

    const handleEditSession = (session) => {
        setCurrentSession(session);
        setSessionSubjectId(session.subjectId || '');
        setSessionDate(formatDateToYYYYMMDD(session.date instanceof Date ? session.date : (session.date && session.date.toDate ? session.date.toDate() : new Date())));
        setSessionDuration(session.duration ? session.duration.toString() : '');
        setSessionType(session.type);
        setSessionDescription(session.description || '');
        setSessionStartTime(session.startTime || '');
        setSessionEndTime(session.endTime || '');
        setIsEditSessionModal(true);
    };

    const handleUpdateSession = async () => {
        if (!currentSession || (!sessionSubjectId && sessionType === 'study')) {
            setMessage('Debe seleccionar una asignatura para la sesión de estudio.');
            setMessageType('error');
            return;
        }
        if (!sessionDate) {
            setMessage('Debe seleccionar una fecha para la sesión.');
            setMessageType('error');
            return;
        }
        if (sessionType === 'study' && !sessionDuration && (!sessionStartTime || !sessionEndTime)) {
            setMessage('Debe especificar la duración o las horas de inicio/fin para la sesión de estudio.');
            setMessageType('error');
            return;
        }
        if (sessionType !== 'study' && !sessionDescription.trim()) {
            setMessage('Debe añadir una descripción para el evento académico.');
            setMessageType('error');
            return;
        }

        let calculatedDuration = parseInt(sessionDuration, 10) || 0;
        let sessionDateTime = normalizeDate(new Date(sessionDate));

        if (sessionStartTime && sessionEndTime) {
            const startMs = new Date(`${sessionDate}T${sessionStartTime}:00`).getTime();
            let endMs = new Date(`${sessionDate}T${sessionEndTime}:00`).getTime();
            if (endMs < startMs) { // Handle overnight sessions
                const nextDayEndDt = new Date(`${sessionDate}T${sessionEndTime}:00`);
                nextDayEndDt.setDate(nextDayEndDt.getDate() + 1);
                endMs = nextDayEndDt.getTime();
            }
            calculatedDuration = Math.round((endMs - startMs) / (1000 * 60));
        }


        const sessionData = {
            subjectId: sessionSubjectId || null,
            date: sessionDateTime,
            duration: calculatedDuration,
            type: sessionType,
            description: sessionDescription.trim(),
            startTime: sessionStartTime || null,
            endTime: sessionEndTime || null
        };

        const result = await updateStudySession(currentSession.id, sessionData);
        if (result.success) {
            setMessage('Sesión/Evento actualizado con éxito.');
            setMessageType('success');
            setIsEditSessionModal(false);
            resetSessionForm();
        } else {
            setMessage(`Error al actualizar sesión/evento: ${result.error}`);
            setMessageType('error');
        }
    };

    const handleDeleteSessionConfirmed = async (id) => {
        const result = await deleteStudySession(id);
        if (result.success) {
            setMessage('Sesión/Evento eliminado con éxito.');
            setMessageType('success');
        } else {
            setMessage(`Error al eliminar sesión/evento: ${result.error}`);
            setMessageType('error');
        }
    };

    const handleDeleteSession = (id) => {
        openConfirmModal(
            'Confirmar Eliminación',
            '¿Estás seguro de que quieres eliminar esta sesión/evento?',
            () => handleDeleteSessionConfirmed(id)
        );
    };

    return (
        <div className="p-4 sm:p-6 lg:p-8 bg-white text-gray-900 min-h-[calc(100vh-64px)] rounded-lg shadow-inner border border-gray-200 dark:bg-gray-900 dark:text-white dark:border-gray-700">
            <h1 className="text-3xl font-bold mb-6 text-center text-blue-700 dark:text-blue-400">Calendario de Estudio</h1>
            <p className="text-center text-sm mb-6 text-gray-600 dark:text-gray-400">ID de Usuario: {userId}</p>

            {message && (
                <div className={`p-3 mb-4 rounded-lg text-center text-sm ${messageType === 'success' ? 'bg-green-600 dark:bg-green-700' : 'bg-red-600 dark:bg-red-700'} text-white`}>
                    {message}
                </div>
            )}

            <div className="flex justify-between items-center mb-6 flex-wrap gap-3">
                <div className="flex items-center space-x-2 w-full sm:w-auto justify-center sm:justify-start">
                    <button onClick={handlePrev} className="bg-gray-200 hover:bg-gray-300 text-gray-800 p-2 rounded-full dark:bg-gray-700 dark:hover:bg-gray-600 dark:text-white">
                        <ChevronLeft size={20} />
                    </button>
                    <span className="text-xl font-semibold text-gray-900 dark:text-white">
                        {view === 'monthly' ? getMonthName(currentDate) :
                            (view === 'weekly' ? `Semana de ${getStartOfWeek(currentDate).getDate()} ${getStartOfWeek(currentDate).toLocaleString('es-ES', { month: 'short' })}` :
                                `${getDayName(currentDate)}, ${currentDate.getDate()} de ${currentDate.toLocaleString('es-ES', { month: 'long' })}`)}
                    </span>
                    <button onClick={handleNext} className="bg-gray-200 hover:bg-gray-300 text-gray-800 p-2 rounded-full dark:bg-gray-700 dark:hover:bg-gray-600 dark:text-white">
                        <ChevronRight size={20} />
                    </button>
                </div>
                <div className="flex space-x-2 w-full sm:w-auto justify-center sm:justify-end">
                    <button
                        onClick={() => setView('daily')}
                        className={`py-2 px-4 rounded-md text-sm font-medium ${view === 'daily' ? 'bg-blue-600 text-white dark:bg-blue-700' : 'bg-gray-200 text-gray-800 hover:bg-gray-300 dark:bg-gray-700 dark:text-gray-300 dark:hover:bg-gray-600'}`}
                    >
                        Diario
                    </button>
                    <button
                        onClick={() => setView('weekly')}
                        className={`py-2 px-4 rounded-md text-sm font-medium ${view === 'weekly' ? 'bg-blue-600 text-white dark:bg-blue-700' : 'bg-gray-200 text-gray-800 hover:bg-gray-300 dark:bg-gray-700 dark:text-gray-300 dark:hover:bg-gray-600'}`}
                    >
                        Semanal
                    </button>
                    <button
                        onClick={() => setView('monthly')}
                        className={`py-2 px-4 rounded-md text-sm font-medium ${view === 'monthly' ? 'bg-blue-600 text-white dark:bg-blue-700' : 'bg-gray-200 text-gray-800 hover:bg-gray-300 dark:bg-gray-700 dark:text-gray-300 dark:hover:bg-gray-600'}`}
                    >
                        Mensual
                    </button>
                </div>
            </div>

            <button
                onClick={() => {
                    resetSessionForm();
                    setSessionDate(formatDateToYYYYMMDD(currentDate)); // Set to current view date
                    setIsAddSessionModalOpen(true);
                }}
                className="w-full bg-blue-600 hover:bg-blue-700 text-white font-bold py-3 px-4 rounded-lg shadow-md transition duration-300 ease-in-out transform hover:scale-105 flex items-center justify-center mb-6 dark:bg-blue-700 dark:hover:bg-blue-800"
            >
                <Plus size={20} className="mr-2" />
                Añadir Nueva Sesión/Evento
            </button>

            {view === 'monthly' && renderMonthlyView()}
            {view === 'weekly' && renderWeeklyView()}
            {view === 'daily' && renderDailyView()}

            {/* Add Session/Event Modal */}
            <Modal title="Añadir Sesión o Evento" isOpen={isAddSessionModalOpen} onClose={() => setIsAddSessionModalOpen(false)}>
                <div className="flex flex-col space-y-4">
                    {message && (
                        <div className={`p-2 rounded text-sm ${messageType === 'success' ? 'bg-green-600 dark:bg-green-700' : 'bg-red-600 dark:bg-red-700'} text-white`}>
                            {message}
                        </div>
                    )}
                    <select
                        value={sessionType}
                        onChange={(e) => {
                            setSessionType(e.target.value);
                            // Reset related fields if type changes
                            if (e.target.value !== 'study') {
                                setSessionSubjectId('');
                                setSessionDuration('');
                            } else {
                                setSessionDescription('');
                            }
                        }}
                        className="p-3 bg-gray-50 text-gray-900 rounded-md border border-gray-300 focus:ring-blue-500 focus:border-blue-500 dark:bg-gray-700 dark:text-white dark:border-gray-600"
                    >
                        <option value="study">Sesión de Estudio</option>
                        <option value="exam">Examen</option>
                        <option value="assignment">Entrega de Trabajo</option>
                        <option value="class">Clase</option>
                        <option value="other">Otro Evento</option>
                    </select>

                    {/* Subject selection visible for all types now */}
                    <select
                        value={sessionSubjectId}
                        onChange={(e) => setSessionSubjectId(e.target.value)}
                        className="p-3 bg-gray-50 text-gray-900 rounded-md border border-gray-300 focus:ring-blue-500 focus:border-blue-500 dark:bg-gray-700 dark:text-white dark:border-gray-600"
                    >
                        <option value="">Selecciona una Asignatura (Opcional)</option>
                        {subjects.map(sub => (
                            <option key={sub.id} value={sub.id}>{sub.name}</option>
                        ))}
                    </select>

                    <input
                        type="date"
                        value={sessionDate}
                        onChange={(e) => setSessionDate(e.target.value)}
                        className="p-3 bg-gray-50 text-gray-900 rounded-md border border-gray-300 focus:ring-blue-500 focus:border-blue-500 dark:bg-gray-700 dark:text-white dark:border-gray-600"
                    />

                    <div className="flex space-x-2">
                        <input
                            type="time"
                            placeholder="Hora de inicio"
                            value={sessionStartTime}
                            onChange={(e) => setSessionStartTime(e.target.value)}
                            className="flex-1 p-3 bg-gray-50 text-gray-900 rounded-md border border-gray-300 focus:ring-blue-500 focus:border-blue-500 dark:bg-gray-700 dark:text-white dark:border-gray-600"
                        />
                        <input
                            type="time"
                            placeholder="Hora de fin"
                            value={sessionEndTime}
                            onChange={(e) => setSessionEndTime(e.target.value)}
                            className="flex-1 p-3 bg-gray-50 text-gray-900 rounded-md border border-gray-300 focus:ring-blue-500 focus:border-blue-500 dark:bg-gray-700 dark:text-white dark:border-gray-600"
                        />
                    </div>

                    {sessionType === 'study' && (
                        <input
                            type="number"
                            placeholder="Duración en minutos (ej: 60)"
                            value={sessionDuration}
                            onChange={(e) => setSessionDuration(e.target.value)}
                            className="p-3 bg-gray-50 text-gray-900 rounded-md border border-gray-300 focus:ring-blue-500 focus:border-blue-500 dark:bg-gray-700 dark:text-white dark:border-gray-600"
                        />
                    )}

                    {(sessionType !== 'study' || (sessionType === 'study' && !sessionStartTime && !sessionEndTime)) && (
                        <input
                            type="text"
                            placeholder="Descripción del evento (ej: Examen de Álgebra)"
                            value={sessionDescription}
                            onChange={(e) => setSessionDescription(e.target.value)}
                            className="p-3 bg-gray-50 text-gray-900 rounded-md border border-gray-300 focus:ring-blue-500 focus:border-blue-500 dark:bg-gray-700 dark:text-white dark:border-gray-600"
                        />
                    )}

                    <button
                        onClick={handleAddSession}
                        className="bg-blue-600 hover:bg-blue-700 text-white font-bold py-3 px-4 rounded-md mt-4 dark:bg-blue-700 dark:hover:bg-blue-800"
                    >
                        Guardar Sesión/Evento
                    </button>
                </div>
            </Modal>

            {/* Edit Session/Event Modal */}
            <Modal title="Editar Sesión o Evento" isOpen={isEditSessionModalOpen} onClose={() => setIsEditSessionModal(false)}>
                <div className="flex flex-col space-y-4">
                    {message && (
                        <div className={`p-2 rounded text-sm ${messageType === 'success' ? 'bg-green-600 dark:bg-green-700' : 'bg-red-600 dark:bg-red-700'} text-white`}>
                            {message}
                        </div>
                    )}
                    <select
                        value={sessionType}
                        onChange={(e) => {
                            setSessionType(e.target.value);
                            if (e.target.value !== 'study') {
                                setSessionSubjectId('');
                                setSessionDuration('');
                            } else {
                                setSessionDescription('');
                            }
                        }}
                        className="p-3 bg-gray-50 text-gray-900 rounded-md border border-gray-300 focus:ring-blue-500 focus:border-blue-500 dark:bg-gray-700 dark:text-white dark:border-gray-600"
                    >
                        <option value="study">Sesión de Estudio</option>
                        <option value="exam">Examen</option>
                        <option value="assignment">Entrega de Trabajo</option>
                        <option value="class">Clase</option>
                        <option value="other">Otro Evento</option>
                    </select>

                    {/* Subject selection visible for all types now */}
                    <select
                        value={sessionSubjectId}
                        onChange={(e) => setSessionSubjectId(e.target.value)}
                        className="p-3 bg-gray-50 text-gray-900 rounded-md border border-gray-300 focus:ring-blue-500 focus:border-blue-500 dark:bg-gray-700 dark:text-white dark:border-gray-600"
                    >
                        <option value="">Selecciona una Asignatura (Opcional)</option>
                        {subjects.map(sub => (
                            <option key={sub.id} value={sub.id}>{sub.name}</option>
                        ))}
                    </select>

                    <input
                        type="date"
                        value={sessionDate}
                        onChange={(e) => setSessionDate(e.target.value)}
                        className="p-3 bg-gray-50 text-gray-900 rounded-md border border-gray-300 focus:ring-blue-500 focus:border-blue-500 dark:bg-gray-700 dark:text-white dark:border-gray-600"
                    />

                    <div className="flex space-x-2">
                        <input
                            type="time"
                            placeholder="Hora de inicio"
                            value={sessionStartTime}
                            onChange={(e) => setSessionStartTime(e.target.value)}
                            className="flex-1 p-3 bg-gray-50 text-gray-900 rounded-md border border-gray-300 focus:ring-blue-500 focus:border-blue-500 dark:bg-gray-700 dark:text-white dark:border-gray-600"
                        />
                        <input
                            type="time"
                            placeholder="Hora de fin"
                            value={sessionEndTime}
                            onChange={(e) => setSessionEndTime(e.target.value)}
                            className="flex-1 p-3 bg-gray-50 text-gray-900 rounded-md border border-gray-300 focus:ring-blue-500 focus:border-blue-500 dark:bg-gray-700 dark:text-white dark:border-gray-600"
                        />
                    </div>

                    {sessionType === 'study' && (
                        <input
                            type="number"
                            placeholder="Duración en minutos (ej: 60)"
                            value={sessionDuration}
                            onChange={(e) => setSessionDuration(e.target.value)}
                            className="p-3 bg-gray-50 text-gray-900 rounded-md border border-gray-300 focus:ring-blue-500 focus:border-blue-500 dark:bg-gray-700 dark:text-white dark:border-gray-600"
                        />
                    )}

                    {(sessionType !== 'study' || (sessionType === 'study' && !sessionStartTime && !sessionEndTime)) && (
                        <input
                            type="text"
                            placeholder="Descripción del evento (ej: Examen de Álgebra)"
                            value={sessionDescription}
                            onChange={(e) => setSessionDescription(e.target.value)}
                            className="p-3 bg-gray-50 text-gray-900 rounded-md border border-gray-300 focus:ring-blue-500 focus:border-blue-500 dark:bg-gray-700 dark:text-white dark:border-gray-600"
                        />
                    )}

                    <button
                        onClick={handleUpdateSession}
                        className="bg-blue-600 hover:bg-blue-700 text-white font-bold py-3 px-4 rounded-md mt-4 dark:bg-blue-700 dark:hover:bg-blue-800"
                    >
                        Actualizar Sesión/Evento
                    </button>
                    <button
                        onClick={() => handleDeleteSession(currentSession.id)}
                        className="bg-gray-500 hover:bg-gray-600 text-white font-bold py-3 px-4 rounded-md mt-2 dark:bg-gray-700 dark:hover:bg-gray-800" // Changed to gray
                    >
                        Eliminar Sesión/Evento
                    </button>
                </div>
            </Modal>
        </div>
    );
};

// Study Statistics Component
const StudyStatistics = () => {
    const { subjects, studySessions, userId } = useAppContext();
    const [isDarkMode, setIsDarkMode] = useState(document.documentElement.classList.contains('dark'));
    const [expandedSubject, setExpandedSubject] = useState(null); // State to control expanded subject for breakdown

    // State for date range filtering for line chart
    const [startDate, setStartDate] = useState(() => {
        const d = new Date();
        d.setDate(d.getDate() - 30); // Default to last 30 days
        return d.toISOString().split('T')[0];
    });
    const [endDate, setEndDate] = useState(() => new Date().toISOString().split('T')[0]);

    // State for selected subjects for line chart
    const [selectedSubjectsForLineChart, setSelectedSubjectsForLineChart] = useState([]);


    // Listen for changes in dark mode class on html element
    useEffect(() => {
        const observer = new MutationObserver(() => {
            setIsDarkMode(document.documentElement.classList.contains('dark'));
        });
        observer.observe(document.documentElement, { attributes: true, attributeFilter: ['class'] });
        return () => observer.disconnect();
    }, []);

    // Prepare data for "Tiempo de Estudio por Asignatura" (text format with breakdown)
    const studyTimeBySubjectDetailed = useMemo(() => {
        const data = {};
        subjects.forEach(subject => {
            data[subject.id] = {
                id: subject.id,
                name: subject.name,
                totalMinutes: 0,
                color: subject.color || '#6366F1', // Use subject's color or default
                breakdown: {} // { 'Estudio': X, 'Pomodoro': Y, 'Examen': Z }
            };
        });

        studySessions.forEach(session => {
            if (session.subjectId && data[session.subjectId]) {
                const type = session.type === 'study' || session.type === 'pomodoro' ? 'Estudio' : session.type.charAt(0).toUpperCase() + session.type.slice(1);
                data[session.subjectId].totalMinutes += session.duration || 0;
                data[session.subjectId].breakdown[type] = (data[session.subjectId].breakdown[type] || 0) + (session.duration || 0);
            }
        });

        return Object.values(data).filter(item => item.totalMinutes > 0).sort((a, b) => b.totalMinutes - a.totalMinutes);
    }, [studySessions, subjects]);


    // --- Accumulation Line Chart Data ---
    const cumulativeStudyData = useMemo(() => {
        // Ensure study sessions have actual Date objects
        const sessions = studySessions.map(session => ({
            ...session,
            date: session.date instanceof Date ? session.date : (session.date && session.date.toDate ? session.date.toDate() : new Date(0))
        }));

        // Filter sessions by date range and selected subjects
        const filteredSessions = sessions.filter(session => {
            const sessionDate = new Date(session.date);
            const start = new Date(startDate);
            const end = new Date(endDate);
            // Include sessions that fall on the start or end date
            return sessionDate >= start && sessionDate <= end &&
                   (selectedSubjectsForLineChart.length === 0 || selectedSubjectsForLineChart.includes(session.subjectId));
        }).sort((a, b) => a.date.getTime() - b.date.getTime()); // Sort by date

        const dataMap = new Map(); // Map to store cumulative data by date

        let cumulativeTotal = 0;
        const cumulativeBySubject = new Map(subjects.map(s => [s.id, 0]));

        // Iterate through dates to ensure all days in range are represented
        let currentDateIterator = new Date(startDate);
        const lastDate = new Date(endDate);

        while (currentDateIterator <= lastDate) {
            const dateStr = currentDateIterator.toISOString().split('T')[0];
            const sessionsOnDay = filteredSessions.filter(s => s.date.toISOString().split('T')[0] === dateStr);

            // Calculate total for the day and update cumulative sums
            let dayTotalMinutes = 0;
            const dailySubjectMinutes = new Map(subjects.map(s => [s.id, 0]));

            sessionsOnDay.forEach(session => {
                const duration = session.duration || 0;
                dayTotalMinutes += duration;
                dailySubjectMinutes.set(session.subjectId, (dailySubjectMinutes.get(session.subjectId) || 0) + duration);
            });

            cumulativeTotal += dayTotalMinutes;

            const entry = {
                date: dateStr,
                totalMinutes: cumulativeTotal,
            };

            // Update cumulative by subject
            subjects.forEach(sub => {
                const dailyMinutes = dailySubjectMinutes.get(sub.id) || 0;
                const currentSubjectCumulative = cumulativeBySubject.get(sub.id) + dailyMinutes;
                cumulativeBySubject.set(sub.id, currentSubjectCumulative);
                entry[`subject_${sub.id}_minutes`] = currentSubjectCumulative;
            });

            dataMap.set(dateStr, entry);
            currentDateIterator.setDate(currentDateIterator.getDate() + 1);
        }

        // Fill in missing dates if there are gaps (e.g., no sessions on a day)
        const finalData = [];
        currentDateIterator = new Date(startDate);
        while (currentDateIterator <= lastDate) {
            const dateStr = currentDateIterator.toISOString().split('T')[0];
            if (dataMap.has(dateStr)) {
                finalData.push(dataMap.get(dateStr));
            } else {
                // If a day has no data, carry forward the previous cumulative total
                const lastEntry = finalData.length > 0 ? finalData[finalData.length - 1] : { totalMinutes: 0 };
                const missingEntry = {
                    date: dateStr,
                    totalMinutes: lastEntry.totalMinutes,
                };
                subjects.forEach(sub => {
                    missingEntry[`subject_${sub.id}_minutes`] = lastEntry[`subject_${sub.id}_minutes`] || 0;
                });
                finalData.push(missingEntry);
            }
            currentDateIterator.setDate(currentDateIterator.getDate() + 1);
        }

        return finalData;

    }, [studySessions, subjects, startDate, endDate, selectedSubjectsForLineChart]);


    const toggleSubjectExpansion = (subjectId) => {
        setExpandedSubject(prevId => prevId === subjectId ? null : subjectId);
    };

    const handleSubjectSelectionForLineChart = (subjectId) => {
        setSelectedSubjectsForLineChart(prevSelected => {
            if (subjectId === 'all') {
                return prevSelected.length === subjects.length ? [] : subjects.map(s => s.id);
            }
            if (prevSelected.includes(subjectId)) {
                return prevSelected.filter(id => id !== subjectId);
            } else {
                return [...prevSelected, subjectId];
            }
        });
    };

    return (
        <div className="p-4 sm:p-6 lg:p-8 bg-white text-gray-900 min-h-[calc(100vh-64px)] rounded-lg shadow-inner border border-gray-200 dark:bg-gray-900 dark:text-white dark:border-gray-700">
            <h1 className="text-3xl font-bold mb-6 text-center text-blue-700 dark:text-blue-400">Estadísticas de Estudio</h1>
            <p className="text-center text-sm mb-6 text-gray-600 dark:text-gray-400">ID de Usuario: {userId}</p>

            <div className="grid grid-cols-1 gap-8 mb-8"> {/* Adjusted grid layout for single column */}
                {/* Tiempo de Estudio por Asignatura (Text Format with Expandable Breakdown) */}
                <div className="bg-white p-6 rounded-lg shadow-lg border border-gray-200 dark:bg-gray-800 dark:border-gray-700">
                    <h2 className="text-2xl font-semibold mb-4 text-blue-700 dark:text-blue-300">Tiempo de Estudio por Asignatura</h2>
                    {studyTimeBySubjectDetailed.length === 0 ? (
                        <p className="text-gray-600 text-center dark:text-gray-400">No hay datos de estudio para mostrar. ¡Empieza a registrar tus sesiones!</p>
                    ) : (
                        <ul className="space-y-3">
                            {studyTimeBySubjectDetailed.map(subject => (
                                <li key={subject.id} className="flex flex-col bg-gray-50 p-3 rounded-md shadow-sm border border-gray-100 dark:bg-gray-700 dark:border-gray-600">
                                    <div className="flex justify-between items-center cursor-pointer" onClick={() => toggleSubjectExpansion(subject.id)}>
                                        <span className="text-lg font-semibold text-gray-800 dark:text-white" style={{ color: subject.color }}>{subject.name}</span>
                                        <div className="flex items-center">
                                            <span className="text-blue-600 font-bold mr-2 dark:text-blue-200">{subject.totalMinutes} min</span>
                                            {expandedSubject === subject.id ? <ChevronUp size={18} className="text-gray-500 dark:text-gray-400" /> : <ChevronDown size={18} className="text-gray-500 dark:text-gray-400" />}
                                        </div>
                                    </div>
                                    {expandedSubject === subject.id && (
                                        <ul className="ml-4 mt-2 space-y-1 text-gray-700 dark:text-gray-300 text-sm">
                                            {Object.entries(subject.breakdown).map(([type, minutes]) => (
                                                <li key={`${subject.name}-${type}`} className="flex justify-between">
                                                    <span>{type}:</span>
                                                    <span>{minutes} min</span>
                                                </li>
                                            ))}
                                            {Object.keys(subject.breakdown).length === 0 && (
                                                <li>No hay desglose por tipo de tarea.</li>
                                            )}
                                        </ul>
                                    )}
                                </li>
                            ))}
                        </ul>
                    )}
                </div>
            </div>

            {/* --- Gráfico de Línea: Acumulación de Tiempo de Estudio --- */}
            <div className="bg-white p-6 rounded-lg shadow-lg border border-gray-200 dark:bg-gray-800 dark:border-gray-700 mt-8">
                <h2 className="text-2xl font-semibold mb-4 text-blue-700 dark:text-blue-300">Gráfico de Línea: Acumulación de Tiempo de Estudio</h2>
                <div className="mb-4 flex flex-col sm:flex-row gap-4 items-center">
                    <div className="flex flex-col w-full sm:w-auto">
                        <label htmlFor="startDate" className="text-gray-700 text-sm mb-1 dark:text-gray-300">Desde:</label>
                        <input
                            type="date"
                            id="startDate"
                            value={startDate}
                            onChange={(e) => setStartDate(e.target.value)}
                            className="p-2 border border-gray-300 rounded-md bg-gray-50 dark:bg-gray-700 dark:border-gray-600 dark:text-white"
                        />
                    </div>
                    <div className="flex flex-col w-full sm:w-auto">
                        <label htmlFor="endDate" className="text-gray-700 text-sm mb-1 dark:text-gray-300">Hasta:</label>
                        <input
                            type="date"
                            id="endDate"
                            value={endDate}
                            onChange={(e) => setEndDate(e.target.value)}
                            className="p-2 border border-gray-300 rounded-md bg-gray-50 dark:bg-gray-700 dark:border-gray-600 dark:text-white"
                        />
                    </div>
                    <div className="flex-grow w-full">
                        <label className="text-gray-700 text-sm mb-1 dark:text-gray-300">Seleccionar Asignaturas:</label>
                        <div className="grid grid-cols-2 sm:grid-cols-3 gap-2">
                            <label className="flex items-center space-x-2 text-sm bg-gray-100 p-2 rounded-md dark:bg-gray-700 cursor-pointer">
                                <input
                                    type="checkbox"
                                    checked={selectedSubjectsForLineChart.length === subjects.length}
                                    onChange={() => handleSubjectSelectionForLineChart('all')}
                                    className="form-checkbox text-blue-600"
                                />
                                <span className="dark:text-gray-200">Todas</span>
                            </label>
                            {subjects.map(subject => (
                                <label key={subject.id} className="flex items-center space-x-2 text-sm bg-gray-100 p-2 rounded-md dark:bg-gray-700 cursor-pointer">
                                    <input
                                        type="checkbox"
                                        value={subject.id}
                                        checked={selectedSubjectsForLineChart.includes(subject.id)}
                                        onChange={() => handleSubjectSelectionForLineChart(subject.id)}
                                        className="form-checkbox text-blue-600"
                                    />
                                    <span style={{ color: subject.color || '#6366F1' }} className="font-medium">{subject.name}</span>
                                        </label>
                            ))}
                        </div>
                    </div>
                </div>

                {cumulativeStudyData.length === 0 ? (
                    <p className="text-gray-600 text-center dark:text-gray-400">No hay datos suficientes en el rango de fechas seleccionado para este gráfico.</p>
                ) : (
                    <ResponsiveContainer width="100%" height={400}>
                        <LineChart
                            data={cumulativeStudyData}
                            margin={{ top: 40, right: 30, left: 20, bottom: 60 }} // Increased bottom margin
                        >
                            <CartesianGrid strokeDasharray="3 3" stroke={isDarkMode ? '#4B5563' : '#E5E7EB'} />
                            <XAxis dataKey="date" stroke={isDarkMode ? '#9CA3AF' : '#4B5563'} angle={-45} textAnchor="end" height={80} /> {/* Increased height */}
                            {/* Removed label prop from YAxis */}
                            <YAxis stroke={isDarkMode ? '#9CA3AF' : '#4B5563'} />
                            <Tooltip
                                contentStyle={{ backgroundColor: isDarkMode ? '#374151' : '#F9FAFB', border: '1px solid ' + (isDarkMode ? '#4B5563' : '#D1D5DB'), borderRadius: '8px' }}
                                itemStyle={{ color: isDarkMode ? '#fff' : '#1F2937' }}
                            />
                            <Legend verticalAlign="top" align="center" height={36}/>

                            {selectedSubjectsForLineChart.length === 0 || selectedSubjectsForLineChart.length === subjects.length ? (
                                <Line
                                    type="monotone"
                                    dataKey="totalMinutes"
                                    stroke="#8884d8"
                                    activeDot={{ r: 8 }}
                                    name="Total Acumulado (minutos)" // Changed legend text
                                    dot={false}
                                />
                            ) : (
                                selectedSubjectsForLineChart.map(subjectId => {
                                    const subject = subjects.find(s => s.id === subjectId);
                                    if (!subject) return null;
                                    return (
                                        <Line
                                            key={subject.id}
                                            type="monotone"
                                            dataKey={`subject_${subject.id}_minutes`}
                                            stroke={subject.color || '#82ca9d'}
                                            activeDot={{ r: 8 }}
                                            name={subject.name}
                                            dot={false}
                                        />
                                    );
                                })
                            )}
                        </LineChart>
                    </ResponsiveContainer>
                )}
            </div>
        </div>
    );
};

// Pomodoro Timer Component
const PomodoroTimer = () => {
    const { userId, pomodoroStates, pomodoroActions } = useAppContext();
    const {
        pomodoroMinutes, setPomodoroMinutes,
        breakMinutes, setBreakMinutes,
        longBreakMinutes, setLongBreakMinutes,
        pomodorosBeforeLongBreak, setPomodorosBeforeLongBreak,
        minutes, seconds,
        isActive, isBreak, sessionsCompleted,
        pomodoroMessage, pomodoroMessageType
    } = pomodoroStates;

    const {
        handleStartPausePomodoro,
        handleResetPomodoro,
        setMinutes, setSeconds
    } = pomodoroActions;

    const formatTime = (num) => String(num).padStart(2, '0');

    return (
        <div className="p-4 sm:p-6 lg:p-8 bg-white text-gray-900 min-h-[calc(100vh-64px)] rounded-lg shadow-inner border border-gray-200 flex flex-col items-center justify-center dark:bg-gray-900 dark:text-white dark:border-gray-700">
            <h1 className="text-3xl font-bold mb-6 text-center text-blue-700 dark:text-blue-400">Temporizador Pomodoro</h1>
            <p className="text-center text-sm mb-6 text-gray-600 dark:text-gray-400">ID de Usuario: {userId}</p>

            {pomodoroMessage && (
                <div className={`p-3 mb-4 rounded-lg text-center text-sm ${pomodoroMessageType === 'success' ? 'bg-green-600 dark:bg-green-700' : (pomodoroMessageType === 'error' ? 'bg-red-600 dark:bg-red-700' : 'bg-blue-600 dark:bg-blue-700')} text-white w-full max-w-sm`}>
                    {pomodoroMessage}
                </div>
            )}

            <div className="bg-white p-8 rounded-full shadow-2xl border border-gray-200 flex items-center justify-center mb-8 w-64 h-64 sm:w-80 sm:h-80 relative dark:bg-gray-800 dark:border-gray-700">
                <div className="text-6xl sm:text-7xl font-mono text-blue-700 dark:text-blue-300">
                    {formatTime(minutes)}:{formatTime(seconds)}
                </div>
                {/* Removed the text "En Pomodoro" / "En descanso" */}
            </div>

            <div className="flex space-x-4 mb-8">
                <button
                    onClick={handleStartPausePomodoro}
                    className={`px-6 py-3 rounded-lg font-bold text-lg shadow-md transition duration-300 ease-in-out transform hover:scale-105 ${isActive ? 'bg-blue-600 hover:bg-blue-700 dark:bg-blue-700 dark:hover:bg-blue-800' : 'bg-blue-600 hover:bg-blue-700 dark:bg-blue-700 dark:hover:bg-blue-800'} text-white`}
                >
                    {isActive ? 'Pausar' : 'Iniciar'}
                </button>
                <button
                    onClick={handleResetPomodoro}
                    className="px-6 py-3 rounded-lg bg-gray-500 hover:bg-gray-600 text-white font-bold text-lg shadow-md transition duration-300 ease-in-out transform hover:scale-105 dark:bg-gray-700 dark:hover:bg-gray-800"
                >
                    Reiniciar
                </button>
            </div>

            <div className="w-full max-w-sm flex flex-col space-y-4 mb-8">
                {/* Removed subject selection dropdown */}

                <div className="flex items-center space-x-4 text-gray-700 dark:text-gray-300 bg-gray-100 p-3 rounded-lg border border-gray-200 dark:bg-gray-700 dark:border-gray-600">
                    <label className="flex-1 text-sm font-medium">Pomodoro (minutos):</label>
                    <input
                        type="number"
                        value={pomodoroMinutes}
                        onChange={(e) => {
                            const val = Math.max(1, parseInt(e.target.value, 10));
                            setPomodoroMinutes(val);
                            if (!isActive && !isBreak) setMinutes(val);
                        }}
                        className="w-20 p-2 bg-white text-gray-900 rounded-md border border-gray-300 text-center focus:ring-blue-500 focus:border-blue-500 dark:bg-gray-800 dark:text-white dark:border-gray-600"
                        disabled={isActive}
                    />
                </div>
                <div className="flex items-center space-x-4 text-gray-700 dark:text-gray-300 bg-gray-100 p-3 rounded-lg border border-gray-200 dark:bg-gray-700 dark:border-gray-600">
                    <label className="flex-1 text-sm font-medium">Descanso Corto (minutos):</label>
                    <input
                        type="number"
                        value={breakMinutes}
                        onChange={(e) => setBreakMinutes(Math.max(1, parseInt(e.target.value, 10)))}
                        className="w-20 p-2 bg-white text-gray-900 rounded-md border border-gray-300 text-center focus:ring-blue-500 focus:border-blue-500 dark:bg-gray-800 dark:text-white dark:border-gray-600"
                        disabled={isActive}
                    />
                </div>
                <div className="flex items-center space-x-4 text-gray-700 dark:text-gray-300 bg-gray-100 p-3 rounded-lg border border-gray-200 dark:bg-gray-700 dark:border-gray-600">
                    <label className="flex-1 text-sm font-medium">Descanso Largo (minutos):</label>
                    <input
                        type="number"
                        value={longBreakMinutes}
                        onChange={(e) => setLongBreakMinutes(Math.max(1, parseInt(e.target.value, 10)))}
                        className="w-20 p-2 bg-white text-gray-900 rounded-md border border-gray-300 text-center focus:ring-blue-500 focus:border-blue-500 dark:bg-gray-800 dark:text-white dark:border-gray-600"
                        disabled={isActive}
                    />
                </div>
                {/* New input for pomodoros before long break */}
                <div className="flex items-center space-x-4 text-gray-700 dark:text-gray-300 bg-gray-100 p-3 rounded-lg border border-gray-200 dark:bg-gray-700 dark:border-gray-600">
                    <label className="flex-1 text-sm font-medium">Pomodoros antes descanso largo:</label>
                    <input
                        type="number"
                        value={pomodorosBeforeLongBreak}
                        onChange={(e) => setPomodorosBeforeLongBreak(Math.max(1, parseInt(e.target.value, 10)))}
                        className="w-20 p-2 bg-white text-gray-900 rounded-md border border-gray-300 text-center focus:ring-blue-500 focus:border-blue-500 dark:bg-gray-800 dark:text-white dark:border-gray-600"
                        disabled={isActive}
                    />
                </div>
            </div>

            <p className="text-gray-600 text-sm mt-4 dark:text-gray-400">Sesiones Pomodoro completadas: {sessionsCompleted}</p>
        </div>
    );
};


// Main App Component
const App = () => {
    const [activeTab, setActiveTab] = useState('subjects'); // 'subjects', 'todo', 'calendar', 'stats', 'timer'
    const [isDarkMode, setIsDarkMode] = useState(() => {
        // Initialize from localStorage or default to true (dark mode)
        const savedMode = localStorage.getItem('uniStudyAppDarkMode');
        return savedMode === 'true' || savedMode === null; // Default to dark if not set
    });

    useEffect(() => {
        // Apply or remove 'dark' class to html element
        if (isDarkMode) {
            document.documentElement.classList.add('dark');
        } else {
            document.documentElement.classList.remove('dark');
        }
        // Save preference to localStorage
        localStorage.setItem('uniStudyAppDarkMode', isDarkMode);
    }, [isDarkMode]);

    const toggleDarkMode = () => {
        setIsDarkMode(prevMode => !prevMode);
    };

    const renderContent = () => {
        switch (activeTab) {
            case 'subjects':
                return <SubjectManagement />;
            case 'todo':
                return <TodoList />;
            case 'calendar':
                return <CalendarView />;
            case 'stats':
                return <StudyStatistics />;
            case 'timer':
                return <PomodoroTimer />;
            default:
                return <SubjectManagement />;
        }
    };

    return (
        <AppProvider>
            <div className="min-h-screen bg-gray-100 font-sans text-gray-900 antialiased flex flex-col dark:bg-gray-950 dark:text-white">
                {/* Navbar */}
                <nav className="bg-white p-4 shadow-lg border-b border-gray-200 dark:bg-gray-800 dark:border-gray-700">
                    <div className="container mx-auto flex flex-col sm:flex-row justify-between items-center">
                        <h1 className="text-2xl font-bold text-blue-700 mb-4 sm:mb-0 dark:text-blue-400">MyTracker</h1>
                        <div className="flex space-x-2 sm:space-x-4 flex-wrap justify-center">
                            <button
                                onClick={() => setActiveTab('subjects')}
                                className={`flex items-center px-4 py-2 rounded-md transition-colors duration-200 ${activeTab === 'subjects' ? 'bg-blue-600 text-white dark:bg-blue-700' : 'text-gray-600 hover:bg-gray-200 dark:text-gray-300 dark:hover:bg-gray-700'}`}
                            >
                                <BookOpen size={20} className="mr-2" />
                                Asignaturas
                            </button>
                            <button
                                onClick={() => setActiveTab('todo')}
                                className={`flex items-center px-4 py-2 rounded-md transition-colors duration-200 ${activeTab === 'todo' ? 'bg-blue-600 text-white dark:bg-blue-700' : 'text-gray-600 hover:bg-gray-200 dark:text-gray-300 dark:hover:bg-gray-700'}`}
                            >
                                <CheckCircle size={20} className="mr-2" />
                                To-do list
                            </button>
                            <button
                                onClick={() => setActiveTab('calendar')}
                                className={`flex items-center px-4 py-2 rounded-md transition-colors duration-200 ${activeTab === 'calendar' ? 'bg-blue-600 text-white dark:bg-blue-700' : 'text-gray-600 hover:bg-gray-200 dark:text-gray-300 dark:hover:bg-gray-700'}`}
                            >
                                <Calendar size={20} className="mr-2" />
                                Calendario
                            </button>
                            <button
                                onClick={() => setActiveTab('stats')}
                                className={`flex items-center px-4 py-2 rounded-md transition-colors duration-200 ${activeTab === 'stats' ? 'bg-blue-600 text-white dark:bg-blue-700' : 'text-gray-600 hover:bg-gray-200 dark:text-gray-300 dark:hover:bg-gray-700'}`}
                            >
                                <BarChart2 size={20} className="mr-2" />
                                Estadísticas
                            </button>
                            <button
                                onClick={() => setActiveTab('timer')}
                                className={`flex items-center px-4 py-2 rounded-md transition-colors duration-200 ${activeTab === 'timer' ? 'bg-blue-600 text-white dark:bg-blue-700' : 'text-gray-600 hover:bg-gray-200 dark:text-gray-300 dark:hover:bg-gray-700'}`}
                            >
                                <Timer size={20} className="mr-2" />
                                Pomodoro
                            </button>
                            <button
                                onClick={toggleDarkMode}
                                className="flex items-center px-4 py-2 rounded-md transition-colors duration-200 text-gray-600 hover:bg-gray-200 dark:text-gray-300 dark:hover:bg-gray-700"
                                title={isDarkMode ? 'Cambiar a Modo Claro' : 'Cambiar a Modo Oscuro'}
                            >
                                {isDarkMode ? <Sun size={20} className="mr-2" /> : <Moon size={20} className="mr-2" />}
                                {isDarkMode ? 'Claro' : 'Oscuro'}
                            </button>
                        </div>
                    </div>
                </nav>

                {/* Main Content */}
                <main className="flex-1 container mx-auto p-4 py-8">
                    {renderContent()}
                </main>
            </div>
        </AppProvider>
    );
};

export default App;
