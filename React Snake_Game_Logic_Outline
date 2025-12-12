import React, { useState, useEffect, useRef, useCallback, useMemo } from 'react';
import { User, LogIn, LogOut, Swords, ListOrdered, Eye } from 'lucide-react';

// --- Configuration Constants ---
const GRID_SIZE = 20;
const INITIAL_SPEED_MS = 180;
const SPEED_INCREMENT = 10;
const INITIAL_SNAKE = [
  [10, 10], // Head
  [10, 9],
  [10, 8],
];

// --- Mock Backend Service ---
// This centralizes all "API calls" as requested, returning Promises
// for easy integration with async/await patterns.
const MockApiService = {
  // Mock database
  users: [
    { id: 'user-1', username: 'playerOne', password: 'p1', score: 15 },
    { id: 'user-2', username: 'proSnake', password: 'p2', score: 25 },
  ],
  leaderboardData: [
    { username: 'GlobalHero', score: 120, rank: 1 },
    { username: 'proSnake', score: 95, rank: 2 },
    { username: 'RattleMaster', score: 78, rank: 3 },
    { username: 'ViperX', score: 55, rank: 4 },
  ],

  // --- Auth Mockups ---
  login: (username, password) => new Promise((resolve, reject) => {
    setTimeout(() => {
      const user = MockApiService.users.find(u => u.username === username && u.password === password);
      if (user) {
        resolve({ id: user.id, username: user.username });
      } else {
        reject('Invalid username or password.');
      }
    }, 500);
  }),

  signup: (username, password) => new Promise((resolve, reject) => {
    setTimeout(() => {
      if (MockApiService.users.some(u => u.username === username)) {
        return reject('Username already taken.');
      }
      const newUser = { id: `user-${Date.now()}`, username, password, score: 0 };
      MockApiService.users.push(newUser);
      resolve({ id: newUser.id, username: newUser.username });
    }, 500);
  }),
  
  logout: () => new Promise((resolve) => {
      setTimeout(() => { resolve(true); }, 200);
  }),

  // --- Multiplayer/Leaderboard Mockups ---
  getLeaderboard: () => new Promise((resolve) => {
    setTimeout(() => {
      resolve(MockApiService.leaderboardData);
    }, 800);
  }),

  // Mock data for the 'watched' player
  getLiveGameData: () => new Promise((resolve) => {
    // Return a simple, mock path for the watched snake
    const watchedSnake = [
      [5, 5], [5, 4], [5, 3]
    ];
    resolve({ 
        playerId: 'watcher-target', 
        username: 'LivePlayer', 
        snake: watchedSnake, 
        direction: 'RIGHT', 
        score: 10 
    });
  }),
};

// --- Core Testable Logic Functions ---

/**
 * Utility function to get random coordinates for food, ensuring no overlap with the snake.
 * (Testable: Provide a snake array, check if the returned food is within grid bounds 
 * and not in the snake array.)
 */
const getRandomCoordinates = (snake) => {
  let newFood;
  let isOverlap;
  
  do {
    const row = Math.floor(Math.random() * GRID_SIZE);
    const col = Math.floor(Math.random() * GRID_SIZE);
    newFood = [row, col];
    isOverlap = snake.some(segment => segment[0] === newFood[0] && segment[1] === newFood[1]);
  } while (isOverlap);

  return newFood;
};

/**
 * Calculates the next head position based on current direction.
 * (Testable: Provide a head coordinate and direction, verify the new coordinate.)
 */
const calculateNewHead = (head, direction) => {
    switch (direction) {
        case 'RIGHT': return [head[0], head[1] + 1];
        case 'LEFT': return [head[0], head[1] - 1];
        case 'UP': return [head[0] - 1, head[1]];
        case 'DOWN': return [head[0] + 1, head[1]];
        default: return head;
    }
}

/**
 * Checks for collision (Wall or Self) and applies Pass-Through logic if enabled.
 * (Testable: Test different head positions near walls for both 'walls' and 
 * 'pass-through' modes, and test self-collision scenarios.)
 */
const checkCollision = (newHead, snake, mode) => {
    const [r, c] = newHead;

    // 1. Self Collision Check (always true regardless of mode)
    if (snake.some(segment => segment[0] === r && segment[1] === c)) {
        return { isCollision: true, newHead: newHead };
    }

    // 2. Wall Collision Check
    if (r < 0 || r >= GRID_SIZE || c < 0 || c >= GRID_SIZE) {
        if (mode === 'walls') {
            return { isCollision: true, newHead: newHead }; // Game Over
        } else if (mode === 'pass-through') {
            // Apply wrap-around logic
            const wrappedR = (r + GRID_SIZE) % GRID_SIZE;
            const wrappedC = (c + GRID_SIZE) % GRID_SIZE;
            return { isCollision: false, newHead: [wrappedR, wrappedC] };
        }
    }
    
    // 3. No collision
    return { isCollision: false, newHead: newHead };
}


// --- Main App Component ---
export default function App() {
  // --- Auth/User State ---
  const [user, setUser] = useState(null); // { id, username }
  const isLoggedIn = !!user;
  const [authError, setAuthError] = useState('');
  const [authLoading, setAuthLoading] = useState(false);

  // --- Game State (Current Player) ---
  const [snake, setSnake] = useState(INITIAL_SNAKE);
  const [food, setFood] = useState(getRandomCoordinates(INITIAL_SNAKE));
  const [direction, setDirection] = useState('RIGHT');
  const [speed, setSpeed] = useState(INITIAL_SPEED_MS);
  const [score, setScore] = useState(0);
  const [status, setStatus] = useState('IDLE'); // IDLE, RUNNING, GAME_OVER

  // --- UI/Mode State ---
  const [activeView, setActiveView] = useState('GAME'); // GAME, LEADERBOARD, WATCHING, LOGIN
  const [mode, setMode] = useState('walls'); // 'walls' or 'pass-through'

  // --- Watching State ---
  const [watchedPlayer, setWatchedPlayer] = useState(null);
  const [watchedSnake, setWatchedSnake] = useState([]);
  const [watchingLoading, setWatchingLoading] = useState(false);
  
  // Ref to hold the latest state for the interval callback (prevents stale closure)
  const stateRef = useRef({ snake, direction, score, food, mode, status });

  // Update ref whenever core state changes
  useEffect(() => {
    stateRef.current = { snake, direction, score, food, mode, status };
  }, [snake, direction, score, food, mode, status]);
  
  // --- Auth Handlers ---
  const handleAuth = async (action, username, password) => {
    setAuthLoading(true);
    setAuthError('');
    try {
      if (action === 'login') {
        const userData = await MockApiService.login(username, password);
        setUser(userData);
        setActiveView('GAME');
      } else if (action === 'signup') {
        const userData = await MockApiService.signup(username, password);
        setUser(userData);
        setActiveView('GAME');
      }
    } catch (error) {
      setAuthError(error);
    } finally {
      setAuthLoading(false);
    }
  };

  const handleLogout = async () => {
    await MockApiService.logout();
    setUser(null);
    setActiveView('LOGIN');
  }

  // --- Game Loop Function ---
  const moveSnake = useCallback(() => {
    const { snake, direction, food, mode, status } = stateRef.current;
    if (status !== 'RUNNING') return;
    
    const head = snake[0];
    const nextHeadCandidate = calculateNewHead(head, direction);
    
    const { isCollision, newHead } = checkCollision(nextHeadCandidate, snake.slice(1), mode);

    if (isCollision) {
      setStatus('GAME_OVER');
      return;
    }

    // New snake array
    const newSnake = [newHead, ...snake];
    let newScore = stateRef.current.score;
    let newSpeed = speed;
    let newFood = food;

    // Check for Food
    if (newHead[0] === food[0] && newHead[1] === food[1]) {
      // EAT: Grow snake, increase score/speed, new food
      newScore += 1;
      newSpeed = Math.max(50, speed - SPEED_INCREMENT); 
      newFood = getRandomCoordinates(newSnake);
    } else {
      // MOVE: Remove the tail segment
      newSnake.pop();
    }
    
    // State updates
    setSnake(newSnake);
    setScore(newScore);
    setSpeed(newSpeed);
    setFood(newFood);
  }, [speed]);


  // --- Game Loop Effect (setInterval) ---
  useEffect(() => {
    if (status !== 'RUNNING') return;

    const intervalId = setInterval(moveSnake, speed);
    
    // Cleanup interval on status/speed change
    return () => clearInterval(intervalId);
  }, [status, speed, moveSnake]);

  // --- Input Handling Effect (Keyboard) ---
  useEffect(() => {
    const handleKeyDown = (e) => {
      const key = e.key;
      if (['ArrowUp', 'ArrowDown', 'ArrowLeft', 'ArrowRight', ' '].includes(key)) {
        e.preventDefault();
      }

      const { direction, status } = stateRef.current;
      
      // Handle Start/Restart
      if ((key === ' ' || key === 'Enter')) {
          if (status === 'IDLE') {
              setStatus('RUNNING');
          } else if (status === 'GAME_OVER') {
              resetGame();
          }
          return;
      }
      
      if (status !== 'RUNNING') return;

      let newDirection = direction;
      
      // Determine new direction, preventing immediate reversal
      switch (key) {
        case 'ArrowRight':
          if (direction !== 'LEFT') newDirection = 'RIGHT';
          break;
        case 'ArrowLeft':
          if (direction !== 'RIGHT') newDirection = 'LEFT';
          break;
        case 'ArrowUp':
          if (direction !== 'DOWN') newDirection = 'UP';
          break;
        case 'ArrowDown':
          if (direction !== 'UP') newDirection = 'DOWN';
          break;
        default:
          return;
      }
      setDirection(newDirection);
    };

    window.addEventListener('keydown', handleKeyDown);
    return () => window.removeEventListener('keydown', handleKeyDown);
  }, []);

  const resetGame = () => {
    setSnake(INITIAL_SNAKE);
    setDirection('RIGHT');
    setSpeed(INITIAL_SPEED_MS);
    setScore(0);
    setFood(getRandomCoordinates(INITIAL_SNAKE));
    setStatus('IDLE');
  };
  
  // Mock logic for the watched snake's movement (simulated network lag/updates)
  useEffect(() => {
      if (activeView !== 'WATCHING') return;

      setWatchingLoading(true);
      
      const fetchMockGame = async () => {
          try {
              const data = await MockApiService.getLiveGameData();
              setWatchedPlayer(data.username);
              setWatchedSnake(data.snake);
          } catch(e) {
              console.error("Could not fetch watched game data", e);
          } finally {
              setWatchingLoading(false);
          }
      };

      fetchMockGame();
      
      // Simple logic to make the watched snake move locally for visual effect
      const interval = setInterval(() => {
          setWatchedSnake(currentSnake => {
              if (currentSnake.length === 0) return currentSnake;
              const head = currentSnake[0];
              const newHead = calculateNewHead(head, 'RIGHT'); // Simple fixed direction
              const newSnake = [newHead, ...currentSnake.slice(0, currentSnake.length - 1)];
              return newSnake;
          });
      }, 500); // Slower update speed for "network effect"

      return () => clearInterval(interval);

  }, [activeView]);

  // --- Sub-Components (Defined for organization) ---

  const Cell = useMemo(() => ({ row, col, currentSnake, currentFood, isWatched = false }) => {
    const isSnake = currentSnake.some(segment => segment[0] === row && segment[1] === col);
    const isHead = isSnake && currentSnake[0][0] === row && currentSnake[0][1] === col;
    const isFood = currentFood && currentFood[0] === row && currentFood[1] === col;

    let className = 'aspect-square transition-colors duration-50';
    
    if (isHead) {
      className += isWatched ? ' bg-blue-700 rounded-sm' : ' bg-green-700 rounded-sm shadow-md';
    } else if (isSnake) {
      className += isWatched ? ' bg-blue-500 rounded-sm' : ' bg-green-500 rounded-sm';
    } else if (isFood) {
      className += ' flex items-center justify-center bg-red-500 rounded-full scale-90 shadow-lg animate-pulse';
    } else {
      className += ' bg-gray-800/10 hover:bg-gray-800/20';
    }

    return (
        <div className={className} style={{ width: '100%', height: '100%' }}>
            {isFood && (
                <span role="img" aria-label="apple" className="text-sm md:text-base">🍎</span>
            )}
        </div>
    );
  }, []);

  const GameGrid = useMemo(() => ({ currentSnake, currentFood, isWatched = false }) => {
    const rows = Array.from({ length: GRID_SIZE }, (_, r) => r);
    const cols = Array.from({ length: GRID_SIZE }, (_, c) => c);

    return (
      <div 
        className={`grid shadow-2xl p-0.5 rounded-lg border-4 ${isWatched ? 'border-blue-700/80' : 'border-green-700/80'} bg-gray-900/50 backdrop-blur-sm mx-auto`}
        style={{
          gridTemplateColumns: `repeat(${GRID_SIZE}, minmax(0, 1fr))`,
          aspectRatio: '1 / 1',
          maxWidth: isWatched ? '300px' : '500px', // Smaller grid for watching
        }}
      >
        {rows.map(r => 
          cols.map(c => (
            <Cell 
              key={`${r}-${c}`} 
              row={r} 
              col={c} 
              currentSnake={currentSnake} 
              currentFood={currentFood}
              isWatched={isWatched}
            />
          ))
        )}
      </div>
    );
  }, [Cell]);

  const AuthView = () => {
    const [isLogin, setIsLogin] = useState(true);
    const [username, setUsername] = useState('');
    const [password, setPassword] = useState('');

    const handleSubmit = (e) => {
      e.preventDefault();
      handleAuth(isLogin ? 'login' : 'signup', username, password);
    };

    return (
      <div className="flex flex-col items-center justify-center min-h-full p-4">
        <div className="w-full max-w-sm bg-gray-800 p-8 rounded-xl shadow-2xl border border-gray-700">
          <h2 className="text-3xl font-bold text-white mb-6 text-center">{isLogin ? 'Log In' : 'Sign Up'}</h2>
          
          <form onSubmit={handleSubmit} className="space-y-4">
            <input 
              type="text" 
              placeholder="Username" 
              value={username} 
              onChange={(e) => setUsername(e.target.value)}
              className="w-full p-3 rounded-lg bg-gray-700 border border-gray-600 text-white focus:ring-green-500 focus:border-green-500"
              required
            />
            <input 
              type="password" 
              placeholder="Password" 
              value={password} 
              onChange={(e) => setPassword(e.target.value)}
              className="w-full p-3 rounded-lg bg-gray-700 border border-gray-600 text-white focus:ring-green-500 focus:border-green-500"
              required
            />
            {authError && <p className="text-red-400 text-sm">{authError}</p>}
            
            <button
              type="submit"
              disabled={authLoading}
              className={`w-full py-3 rounded-lg text-xl font-bold transition duration-300 shadow-lg ${
                authLoading ? 'bg-gray-500' : 'bg-green-600 hover:bg-green-700'
              }`}
            >
              {authLoading ? 'Loading...' : (isLogin ? 'Log In' : 'Sign Up')}
            </button>
          </form>

          <button
            onClick={() => setIsLogin(!isLogin)}
            className="w-full mt-4 text-sm text-green-400 hover:text-green-300 transition duration-300"
          >
            {isLogin ? "Don't have an account? Sign Up" : "Already have an account? Log In"}
          </button>
        </div>
      </div>
    );
  };
  
  const LeaderboardView = () => {
    const [leaderboard, setLeaderboard] = useState([]);
    const [loading, setLoading] = useState(true);

    useEffect(() => {
        setLoading(true);
        MockApiService.getLeaderboard()
            .then(data => {
                setLeaderboard(data);
            })
            .catch(e => console.error("Leaderboard fetch failed:", e))
            .finally(() => setLoading(false));
    }, []);

    return (
        <div className="max-w-xl mx-auto p-4">
            <h2 className="text-4xl font-extrabold text-center text-green-400 mb-8 flex items-center justify-center">
                <ListOrdered className="w-8 h-8 mr-3" /> Global Leaderboard
            </h2>
            
            <div className="bg-gray-800 rounded-xl shadow-2xl overflow-hidden border border-gray-700">
                <div className="flex bg-gray-700 font-bold text-lg p-4 border-b border-gray-600">
                    <span className="w-1/6 text-center">Rank</span>
                    <span className="w-3/6">Player</span>
                    <span className="w-2/6 text-right">High Score</span>
                </div>
                {loading ? (
                    <div className="text-center p-8 text-gray-400">Loading leaderboard...</div>
                ) : (
                    <ul className="divide-y divide-gray-700">
                        {leaderboard.map((item, index) => (
                            <li key={item.username} className={`flex items-center p-4 transition duration-300 ${index < 3 ? 'bg-yellow-900/30' : 'hover:bg-gray-700'}`}>
                                <span className="w-1/6 text-center font-extrabold text-xl">
                                    {index === 0 ? '🥇' : index === 1 ? '🥈' : index === 2 ? '🥉' : `#${index + 1}`}
                                </span>
                                <span className="w-3/6 font-semibold text-lg">{item.username}</span>
                                <span className="w-2/6 text-right text-green-400 font-bold text-xl">{item.score}</span>
                            </li>
                        ))}
                    </ul>
                )}
            </div>
        </div>
    );
  };

  const WatchingView = () => {
    if (watchingLoading) {
        return <div className="text-center p-10 text-xl text-gray-400">Establishing stream connection...</div>
    }

    return (
        <div className="max-w-4xl mx-auto p-4 flex flex-col items-center">
            <h2 className="text-4xl font-extrabold text-center text-blue-400 mb-8 flex items-center justify-center">
                <Eye className="w-8 h-8 mr-3" /> Watching {watchedPlayer || 'Live Player'}
            </h2>
            
            <p className="text-lg text-gray-300 mb-4">
                Current Score: <span className="text-blue-400 font-bold">{watchedSnake.length - INITIAL_SNAKE.length + 1}</span>
            </p>

            <GameGrid currentSnake={watchedSnake} currentFood={food} isWatched={true} />
            
            <p className="mt-8 text-center text-gray-400">
                (This is a mock live feed demonstrating the structure for remote viewing.)
            </p>
        </div>
    );
  };

  const GameView = () => {
    const isIdle = status === 'IDLE';
    const isGameOver = status === 'GAME_OVER';

    const getButtonText = () => {
        if (isGameOver) return "Restart Game (Space/Enter)";
        if (isIdle) return "Start Game (Space/Enter)";
        return "Game Running...";
    }

    return (
        <div className="flex flex-col items-center p-4">
             {/* Score and Info Bar */}
            <div className="flex justify-between w-full max-w-lg md:max-w-xl lg:max-w-2xl mb-6 p-4 rounded-lg bg-gray-800 shadow-xl border border-gray-700">
                <div className="text-xl font-semibold">
                    Score: <span className="text-green-400">{score}</span>
                </div>
                <div className="text-xl font-semibold capitalize">
                    Mode: <span className="text-yellow-400">{mode}</span>
                </div>
            </div>

            {/* Game Grid Container */}
            <div className="w-full max-w-lg md:max-w-xl lg:max-w-2xl mb-6">
                <GameGrid currentSnake={snake} currentFood={food} />
            </div>

            {/* Controls and Status */}
            <button
                onClick={isGameOver ? resetGame : () => status === 'IDLE' && setStatus('RUNNING')}
                disabled={status === 'RUNNING'}
                className={`
                w-full max-w-lg md:max-w-xl lg:max-w-2xl py-3 rounded-lg text-xl font-bold transition-all duration-300 transform
                ${isGameOver 
                    ? 'bg-red-600 hover:bg-red-700 ring-4 ring-red-400/50 animate-pulse' 
                    : status === 'IDLE' 
                    ? 'bg-green-600 hover:bg-green-700 ring-4 ring-green-400/50 shadow-lg'
                    : 'bg-gray-700 text-gray-400 cursor-not-allowed'
                }
                `}
            >
                {getButtonText()}
            </button>
            
            {/* Mode Selector */}
            <div className="flex space-x-4 mt-4 text-sm font-medium">
                <button
                    onClick={() => setMode('walls')}
                    disabled={status === 'RUNNING'}
                    className={`px-3 py-1 rounded-full transition ${mode === 'walls' ? 'bg-yellow-500 text-gray-900 font-bold' : 'bg-gray-700 hover:bg-gray-600'}`}
                >
                    Walls Mode
                </button>
                <button
                    onClick={() => setMode('pass-through')}
                    disabled={status === 'RUNNING'}
                    className={`px-3 py-1 rounded-full transition ${mode === 'pass-through' ? 'bg-yellow-500 text-gray-900 font-bold' : 'bg-gray-700 hover:bg-gray-600'}`}
                >
                    Pass-Through Mode
                </button>
            </div>


            {/* Game Over Message Overlay */}
            {isGameOver && (
                <div className="fixed inset-0 bg-gray-900/90 backdrop-blur-sm flex items-center justify-center z-10">
                    <div className="bg-gray-800 p-8 rounded-xl shadow-2xl text-center border-4 border-red-500 animate-in fade-in zoom-in">
                        <h2 className="text-6xl font-extrabold text-red-500 mb-4">GAME OVER!</h2>
                        <p className="text-3xl font-semibold text-white mb-6">Final Score: <span className="text-yellow-400">{score}</span></p>
                        <button
                            onClick={resetGame}
                            className="px-6 py-3 bg-red-600 text-white text-xl font-bold rounded-lg hover:bg-red-700 transition duration-300 shadow-xl ring-4 ring-red-400/50"
                        >
                            Play Again
                        </button>
                    </div>
                </div>
            )}
        </div>
    );
  }
  
  // --- Navigation Bar ---
  const NavBar = () => (
      <nav className="w-full bg-gray-800 shadow-xl p-4 flex justify-between items-center border-b border-gray-700">
          <div className="text-2xl font-extrabold text-green-400 flex items-center">
              <Swords className="w-6 h-6 mr-2" /> Snake Arena
          </div>
          
          <div className="flex space-x-3 items-center">
                <button
                    onClick={() => setActiveView('GAME')}
                    className={`flex items-center px-3 py-2 rounded-lg text-sm font-semibold transition ${activeView === 'GAME' ? 'bg-green-600 text-white' : 'text-gray-300 hover:bg-gray-700'}`}
                >
                    <Swords className="w-4 h-4 mr-1" /> Play
                </button>
                <button
                    onClick={() => setActiveView('LEADERBOARD')}
                    className={`flex items-center px-3 py-2 rounded-lg text-sm font-semibold transition ${activeView === 'LEADERBOARD' ? 'bg-green-600 text-white' : 'text-gray-300 hover:bg-gray-700'}`}
                >
                    <ListOrdered className="w-4 h-4 mr-1" /> Leaderboard
                </button>
                <button
                    onClick={() => setActiveView('WATCHING')}
                    className={`flex items-center px-3 py-2 rounded-lg text-sm font-semibold transition ${activeView === 'WATCHING' ? 'bg-green-600 text-white' : 'text-gray-300 hover:bg-gray-700'}`}
                >
                    <Eye className="w-4 h-4 mr-1" /> Watch
                </button>
          </div>

          <div className="flex items-center space-x-4">
              {isLoggedIn ? (
                  <>
                      <div className="flex items-center text-green-400 font-bold">
                          <User className="w-4 h-4 mr-2" /> {user.username}
                      </div>
                      <button 
                          onClick={handleLogout}
                          className="flex items-center px-3 py-2 bg-red-600 text-white rounded-lg text-sm font-semibold hover:bg-red-700 transition"
                      >
                          <LogOut className="w-4 h-4 mr-1" /> Log Out
                      </button>
                  </>
              ) : (
                  <button 
                      onClick={() => setActiveView('LOGIN')}
                      className="flex items-center px-3 py-2 bg-green-600 text-white rounded-lg text-sm font-semibold hover:bg-green-700 transition"
                  >
                      <LogIn className="w-4 h-4 mr-1" /> Log In / Sign Up
                  </button>
              )}
          </div>
      </nav>
  );

  // --- Main Render ---
  return (
    <div className="min-h-screen bg-gray-900 flex flex-col font-inter text-white">
      <NavBar />
      
      <main className="flex-grow flex items-center justify-center p-4">
          {activeView === 'GAME' && <GameView />}
          {activeView === 'LEADERBOARD' && <LeaderboardView />}
          {activeView === 'WATCHING' && <WatchingView />}
          {activeView === 'LOGIN' && <AuthView />}
      </main>

      <footer className="p-2 text-center text-sm text-gray-500 border-t border-gray-800">
          <p>Controls: Arrow Keys to move. Space/Enter to start/restart.</p>
      </footer>
    </div>
  );
}
