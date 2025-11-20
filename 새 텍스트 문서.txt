import React, { useState, useEffect, useCallback } from 'react';
import { Home, Search, User, Bell, ChevronLeft, Calendar, Award, CheckCircle, X, Loader2, Lock, ShieldAlert, LogOut, ArrowRight } from 'lucide-react';

// ==================================================================================
// 🔒 [Server-Side Logic] - 보안 백엔드 시뮬레이션
// ==================================================================================

const BackendService = (() => {
  // 내부 데이터 (Private Scope)
  const _SECURE_DB = {
    users: {
      'user_20241234': { 
        name: '김울산', 
        dept: '컴퓨터공학부', 
        realStudentId: '20241234', 
        points: 45, 
        appliedProgramIds: [] 
      }
    },
    programs: [
      {
        id: 1,
        category: '취업/진로',
        title: '2024학년도 하반기 AI 면접 역량 강화 캠프',
        department: '대학일자리플러스센터',
        points: 10,
        date: '2024.11.25 ~ 2024.11.26',
        status: '접수중',
        image: 'https://images.unsplash.com/photo-1521737604893-d14cc237f11d?auto=format&fit=crop&q=80&w=800',
        description: 'AI 면접 트렌드를 분석하고 실제 모의 면접을 통해 취업 역량을 강화하는 프로그램입니다.'
      },
      {
        id: 2,
        category: '학습역량',
        title: '창의적 문제해결(TRIZ) 워크숍',
        department: '교수학습개발센터',
        points: 5,
        date: '2024.11.28',
        status: '접수중',
        image: 'https://images.unsplash.com/photo-1531482615713-2afd69097998?auto=format&fit=crop&q=80&w=800',
        description: '창의적 문제해결 방법론인 TRIZ를 배우고 실생활 문제에 적용해보는 실습형 워크숍입니다.'
      },
      {
        id: 3,
        category: '봉사',
        title: '지역사회와 함께하는 연탄 나눔 봉사',
        department: '사회공헌센터',
        points: 15,
        date: '2024.12.01',
        status: '마감임박',
        image: 'https://images.unsplash.com/photo-1593113598332-cd288d649433?auto=format&fit=crop&q=80&w=800',
        description: '겨울철 난방 취약계층을 위한 연탄 나눔 봉사활동입니다.'
      }
    ]
  };

  const _maskString = (str) => {
    if (!str || str.length < 4) return '****';
    return str.substring(0, 4) + '*'.repeat(str.length - 4);
  };

  return {
    login: async (provider) => {
      return new Promise(resolve => {
        setTimeout(() => {
          // 어떤 방식으로 로그인하든 성공했다고 가정하고 토큰 발급
          resolve({ 
            token: "SECURE_HASH_x8d9f0a1", 
            provider: provider // 로그인한 방식 (uclass, portal 등)
          });
        }, 800);
      });
    },

    getPrograms: async () => {
      return new Promise(resolve => setTimeout(() => resolve([..._SECURE_DB.programs]), 600));
    },

    getUserData: async (token) => {
      return new Promise((resolve, reject) => {
        setTimeout(() => {
          if (token !== "SECURE_HASH_x8d9f0a1") {
            reject(new Error("E001")); 
            return;
          }
          const user = _SECURE_DB.users['user_20241234'];
          
          const safeUser = {
            name: user.name,
            dept: user.dept,
            maskedId: _maskString(user.realStudentId),
            points: user.points,
            appliedProgramIds: [...user.appliedProgramIds]
          };
          
          resolve(safeUser); 
        }, 400);
      });
    },

    applyForProgram: async (token, programId) => {
      return new Promise((resolve, reject) => {
        setTimeout(() => {
          if (token !== "SECURE_HASH_x8d9f0a1") {
            reject(new Error("E002"));
            return;
          }
          const user = _SECURE_DB.users['user_20241234'];
          if (user.appliedProgramIds.includes(programId)) {
            reject(new Error("E003"));
            return;
          }
          user.appliedProgramIds.push(programId);
          resolve({ success: true });
        }, 800);
      });
    }
  };
})();

// ==================================================================================
// 📱 [Client App] - 메인 앱 컴포넌트
// ==================================================================================

export default function App() {
  const [authToken, setAuthToken] = useState(null);
  const [loginProvider, setLoginProvider] = useState(null); // 로그인 방식 저장
  
  const [activeTab, setActiveTab] = useState('home');
  const [selectedProgram, setSelectedProgram] = useState(null);
  const [programs, setPrograms] = useState([]);
  const [userData, setUserData] = useState(null);
  const [userAppliedList, setUserAppliedList] = useState([]);
  
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState(null);
  const [showApplyModal, setShowApplyModal] = useState(false);
  const [isApplying, setIsApplying] = useState(false);

  // 🛡️ 보안: F12 및 우클릭 방지 (유지)
  useEffect(() => {
    const handleContextMenu = (e) => e.preventDefault();
    const handleKeyDown = (e) => {
      if (e.key === 'F12' || (e.ctrlKey && e.shiftKey && (e.key === 'I' || e.key === 'J')) || (e.ctrlKey && e.key === 'U')) {
        e.preventDefault();
      }
    };
    document.addEventListener('contextmenu', handleContextMenu);
    document.addEventListener('keydown', handleKeyDown);
    return () => {
      document.removeEventListener('contextmenu', handleContextMenu);
      document.removeEventListener('keydown', handleKeyDown);
    };
  }, []);

  // 로그인 처리 함수
  const handleLogin = async (provider) => {
    try {
      setLoading(true);
      const loginRes = await BackendService.login(provider);
      setAuthToken(loginRes.token);
      setLoginProvider(provider); // 로그인 방식 저장 (포털/유클래스/클라썸)

      // 로그인 성공 후에만 데이터 로드
      const programList = await BackendService.getPrograms();
      setPrograms(programList);
      await refreshUserData(loginRes.token, programList);
      
      setLoading(false);
    } catch (err) {
      setError("로그인 실패 (Code: AUTH_FAIL)");
      setLoading(false);
    }
  };

  // 로그아웃 처리 함수
  const handleLogout = () => {
    if (window.confirm("정말 로그아웃 하시겠습니까?")) {
      setAuthToken(null);
      setUserData(null);
      setActiveTab('home');
      setLoginProvider(null);
    }
  };

  const refreshUserData = async (token, currentPrograms) => {
    try {
      const data = await BackendService.getUserData(token);
      setUserData(data);
      const myApps = currentPrograms.filter(p => data.appliedProgramIds.includes(p.id));
      setUserAppliedList(myApps);
    } catch (e) {
      console.warn("Sync error");
    }
  };

  const handleSecureApply = async (program) => {
    if (isApplying) return;
    setIsApplying(true);
    try {
      await BackendService.applyForProgram(authToken, program.id);
      alert(`[${program.title}]\n신청이 완료되었습니다.`);
      await refreshUserData(authToken, programs);
      setShowApplyModal(false);
      setSelectedProgram(null);
    } catch (err) {
      if (err.message === "E003") alert("이미 신청 완료된 프로그램입니다.");
      else alert("신청 처리 중 오류가 발생했습니다.");
    } finally {
      setIsApplying(false);
    }
  };

  // 렌더링 로직: 토큰이 없으면 무조건 로그인 화면 출력
  if (!authToken) {
    return <LoginScreen onLogin={handleLogin} isLoading={loading} />;
  }

  const renderScreen = () => {
    if (loading && !programs.length) return <LoadingScreen />;
    if (error) return <ErrorScreen message={error} />;

    switch (activeTab) {
      case 'home': return <HomeScreen programs={programs} onProgramClick={setSelectedProgram} />;
      case 'search': return <SearchScreen programs={programs} onProgramClick={setSelectedProgram} />;
      case 'mypage': return <MyPageScreen userData={userData} appliedList={userAppliedList} onLogout={handleLogout} provider={loginProvider} />;
      default: return <HomeScreen programs={programs} onProgramClick={setSelectedProgram} />;
    }
  };

  return (
    <div className="min-h-screen bg-gray-100 flex justify-center font-sans select-none">
      <div className="w-full max-w-md bg-white min-h-screen shadow-xl relative flex flex-col overflow-hidden">
        
        {/* 메인 헤더 */}
        <header className="bg-emerald-800 text-white p-4 pt-6 flex justify-between items-center sticky top-0 z-10 shadow-md">
          <div className="flex items-center gap-2">
            <h1 className="text-xl font-bold tracking-wider">U-STAR</h1>
            <div className="flex items-center gap-1 bg-emerald-900/50 px-2 py-1 rounded text-[10px] text-emerald-200 border border-emerald-700">
              <Lock size={10} />
              <span>SECURE</span>
            </div>
          </div>
          <div className="relative cursor-pointer hover:opacity-80 transition-opacity">
            <Bell size={24} />
          </div>
        </header>

        <main className="flex-1 overflow-y-auto pb-20 bg-gray-50 scroll-smooth">
          {renderScreen()}
        </main>

        <nav className="bg-white border-t border-gray-200 p-2 flex justify-around items-center absolute bottom-0 w-full h-16 z-20 safe-area-pb">
          <NavButton icon={<Home size={24} />} label="홈" isActive={activeTab === 'home'} onClick={() => setActiveTab('home')} />
          <NavButton icon={<Search size={24} />} label="탐색" isActive={activeTab === 'search'} onClick={() => setActiveTab('search')} />
          <NavButton icon={<User size={24} />} label="마이페이지" isActive={activeTab === 'mypage'} onClick={() => setActiveTab('mypage')} />
        </nav>

        {/* 모달들 */}
        {selectedProgram && (
          <ProgramDetailModal 
            program={selectedProgram} 
            onClose={() => setSelectedProgram(null)} 
            onApply={() => setShowApplyModal(true)}
            isAlreadyApplied={userData?.appliedProgramIds.includes(selectedProgram.id)}
          />
        )}
        {showApplyModal && selectedProgram && (
          <ConfirmModal 
            title="보안 신청"
            content={selectedProgram.title}
            onConfirm={() => handleSecureApply(selectedProgram)}
            onCancel={() => setShowApplyModal(false)}
            isLoading={isApplying}
          />
        )}
      </div>
    </div>
  );
}

// ==================================================================================
// 🔑 [New Component] 로그인 화면 (강제 로그인용)
// ==================================================================================

function LoginScreen({ onLogin, isLoading }) {
  return (
    <div className="min-h-screen bg-emerald-900 flex justify-center items-center p-6 relative overflow-hidden font-sans">
      {/* 배경 장식 */}
      <div className="absolute top-0 left-0 w-full h-full bg-[radial-gradient(circle_at_50%_120%,#10b981,transparent)] opacity-20 pointer-events-none"></div>
      
      <div className="w-full max-w-sm bg-white/10 backdrop-blur-md rounded-3xl p-8 shadow-2xl border border-white/10 flex flex-col items-center">
        <div className="bg-white p-4 rounded-2xl mb-6 shadow-lg">
          <Award size={48} className="text-emerald-700" />
        </div>
        <h1 className="text-3xl font-bold text-white mb-2 tracking-wider">U-STAR</h1>
        <p className="text-emerald-100 text-sm mb-8 text-center">울산대학교 비교과 통합 관리 시스템</p>

        {isLoading ? (
          <div className="py-10 flex flex-col items-center text-emerald-100">
            <Loader2 size={32} className="animate-spin mb-2" />
            <p className="text-xs">안전하게 로그인 중입니다...</p>
          </div>
        ) : (
          <div className="w-full space-y-3">
            <p className="text-white/60 text-xs text-center mb-2">계정 연동 로그인</p>
            
            <button 
              onClick={() => onLogin('portal')}
              className="w-full bg-white hover:bg-gray-50 text-emerald-900 py-3.5 rounded-xl font-bold flex items-center justify-center gap-3 shadow-lg transition-transform active:scale-95"
            >
              <div className="w-5 h-5 rounded-full bg-emerald-700 flex items-center justify-center text-white text-xs">U</div>
              울산대학교 포털 로그인
            </button>

            <button 
              onClick={() => onLogin('uclass')}
              className="w-full bg-[#004EA2] hover:bg-[#003d82] text-white py-3.5 rounded-xl font-bold flex items-center justify-center gap-3 shadow-lg transition-transform active:scale-95"
            >
               <div className="w-5 h-5 rounded-full bg-white/20 flex items-center justify-center text-xs">C</div>
               UCLASS(유클래스) 로그인
            </button>

            <button 
              onClick={() => onLogin('classum')}
              className="w-full bg-[#5244F4] hover:bg-[#4336d6] text-white py-3.5 rounded-xl font-bold flex items-center justify-center gap-3 shadow-lg transition-transform active:scale-95"
            >
              <div className="w-5 h-5 rounded-full bg-white/20 flex items-center justify-center text-xs">T</div>
              CLASSUM(클라썸) 로그인
            </button>

            <p className="text-center text-emerald-200/50 text-[10px] mt-6">
              본 서비스는 울산대학교 학생 인증이 필요합니다.<br/>
              보안을 위해 자동 로그아웃될 수 있습니다.
            </p>
          </div>
        )}
      </div>
    </div>
  );
}


// --- 하위 컴포넌트 수정 (마이페이지 로그아웃 추가) ---

function MyPageScreen({ userData, appliedList, onLogout, provider }) {
  if (!userData) return <div className="p-4">Loading...</div>;

  // 로그인 제공자에 따른 뱃지 텍스트
  const getProviderName = (p) => {
    if (p === 'portal') return '울산대 포털';
    if (p === 'uclass') return 'UCLASS';
    if (p === 'classum') return 'CLASSUM';
    return 'U-STAR';
  };

  return (
    <div className="p-4">
      {/* 프로필 섹션 */}
      <div className="bg-white p-6 rounded-2xl shadow-sm border border-gray-100 mb-6">
        <div className="flex items-center gap-4 mb-4">
          <div className="w-16 h-16 bg-gray-200 rounded-full flex items-center justify-center text-gray-500 relative">
            <User size={32} />
            <div className="absolute bottom-0 right-0 w-5 h-5 bg-emerald-500 border-2 border-white rounded-full"></div>
          </div>
          <div>
            <h2 className="text-xl font-bold text-gray-800">{userData.name} <span className="text-sm font-normal text-gray-500">님</span></h2>
            <p className="text-sm text-emerald-600 font-medium">{userData.dept} {userData.maskedId}</p>
            <span className="text-[10px] bg-gray-100 text-gray-500 px-2 py-0.5 rounded mt-1 inline-block">
              Connected via {getProviderName(provider)}
            </span>
          </div>
        </div>

        {/* 로그아웃 버튼 (요청사항 1) */}
        <button 
          onClick={onLogout}
          className="w-full py-2 border border-gray-200 rounded-xl text-sm font-bold text-gray-500 hover:bg-gray-50 hover:text-red-500 transition-colors flex items-center justify-center gap-2"
        >
          <LogOut size={16} />
          로그아웃
        </button>
      </div>

      <div className="grid grid-cols-2 gap-4 mb-8">
        <div className="bg-emerald-50 p-4 rounded-xl text-center">
          <p className="text-sm text-emerald-800 mb-1">나의 U-STAR 포인트</p>
          <p className="text-2xl font-bold text-emerald-600">{userData.points} P</p>
        </div>
        <div className="bg-purple-50 p-4 rounded-xl text-center">
          <p className="text-sm text-purple-800 mb-1">신청 완료</p>
          <p className="text-2xl font-bold text-purple-600">{appliedList.length} 개</p>
        </div>
      </div>
      
      <div className="flex justify-between items-end mb-3">
        <h3 className="text-lg font-bold text-gray-800 flex items-center gap-2">
          <CheckCircle size={20} className="text-emerald-500" /> 신청 내역
        </h3>
      </div>

      <div className="space-y-3">
        {appliedList.length > 0 ? (
          appliedList.map((program, idx) => (
            <div key={idx} className="bg-white p-4 rounded-xl shadow-sm border border-gray-200 flex justify-between items-center">
               <h4 className="font-bold text-gray-800 text-sm line-clamp-1">{program.title}</h4>
               <span className="text-xs font-bold text-emerald-600 bg-emerald-50 px-2 py-1 rounded">완료</span>
            </div>
          ))
        ) : <p className="text-center text-gray-400 text-sm py-4">신청 내역이 없습니다.</p>}
      </div>
    </div>
  );
}

// --- 나머지 컴포넌트 (기존과 동일하게 유지) ---

function NavButton({ icon, label, isActive, onClick }) {
  return (
    <button onClick={onClick} className={`flex flex-col items-center justify-center w-full h-full transition-colors ${isActive ? 'text-emerald-700' : 'text-gray-400'}`}>
      {icon}
      <span className="text-xs mt-1 font-medium">{label}</span>
    </button>
  );
}

function LoadingScreen() {
  return (
    <div className="flex flex-col items-center justify-center h-full text-gray-400">
      <Loader2 size={48} className="animate-spin mb-4 text-emerald-700" />
      <p className="font-bold text-sm text-emerald-800">데이터 불러오는 중...</p>
    </div>
  );
}

function ErrorScreen({ message }) {
  return (
    <div className="flex flex-col items-center justify-center h-full p-6 text-center">
      <div className="bg-red-100 p-4 rounded-full mb-4 text-red-500"><X size={32}/></div>
      <p className="text-gray-800 font-bold text-sm">{message}</p>
      <p className="text-gray-400 text-xs mt-2">관리자에게 문의하십시오.</p>
    </div>
  );
}

function HomeScreen({ programs, onProgramClick }) {
  return (
    <div className="p-4 space-y-6">
       <div className="bg-gradient-to-r from-emerald-700 to-teal-700 rounded-2xl p-5 text-white shadow-lg flex justify-between items-center">
        <div>
          <p className="text-emerald-100 text-xs mb-1">2024학년도 2학기</p>
          <h2 className="text-xl font-bold">비교과 프로그램 안내</h2>
        </div>
        <ArrowRight className="text-white/50" />
      </div>
      <div className="space-y-4">
        {programs.map(program => (
          <ProgramCard key={program.id} program={program} onClick={() => onProgramClick(program)} />
        ))}
      </div>
    </div>
  );
}

function SearchScreen({ programs, onProgramClick }) {
  const [searchTerm, setSearchTerm] = useState('');
  const filteredPrograms = programs.filter(p => p.title.toLowerCase().includes(searchTerm.toLowerCase()));
  return (
    <div className="p-4">
      <input 
          type="text" 
          placeholder="프로그램 검색" 
          className="w-full p-3 rounded-xl border border-gray-300 mb-4"
          value={searchTerm}
          onChange={(e) => setSearchTerm(e.target.value)}
        />
      <div className="space-y-4">
        {filteredPrograms.map(program => (
          <ProgramCard key={program.id} program={program} onClick={() => onProgramClick(program)} />
        ))}
      </div>
    </div>
  );
}

function ProgramCard({ program, onClick }) {
  return (
    <div onClick={onClick} className="bg-white rounded-xl shadow-sm border border-gray-100 overflow-hidden active:scale-95 transition-transform cursor-pointer">
      <div className="relative h-32 w-full">
        <img src={program.image} alt="program" className="w-full h-full object-cover" />
        <div className="absolute top-2 right-2 bg-white/90 px-2 py-1 rounded text-xs font-bold text-emerald-600">{program.status}</div>
      </div>
      <div className="p-4">
        <h4 className="font-bold text-gray-800 mb-1">{program.title}</h4>
        <p className="text-xs text-gray-500">{program.date} | {program.points}P</p>
      </div>
    </div>
  );
}

function ProgramDetailModal({ program, onClose, onApply, isAlreadyApplied }) {
  return (
    <div className="absolute inset-0 z-50 bg-white flex flex-col animate-in slide-in-from-bottom duration-300">
      <div className="relative h-64 w-full bg-gray-200">
        <img src={program.image} alt="detail" className="w-full h-full object-cover" />
        <button onClick={onClose} className="absolute top-4 left-4 bg-white/80 p-2 rounded-full"><ChevronLeft size={24}/></button>
      </div>
      <div className="flex-1 p-6 overflow-y-auto pb-24">
        <h2 className="text-2xl font-bold mb-2">{program.title}</h2>
        <p className="text-gray-600 text-sm">{program.description}</p>
      </div>
      <div className="absolute bottom-0 w-full p-4 bg-white border-t">
        <button onClick={onApply} disabled={isAlreadyApplied} className={`w-full py-4 rounded-xl font-bold text-white ${isAlreadyApplied ? 'bg-gray-400' : 'bg-emerald-600'}`}>
          {isAlreadyApplied ? '신청 완료됨' : '신청하기'}
        </button>
      </div>
    </div>
  );
}

function ConfirmModal({ title, content, onConfirm, onCancel, isLoading }) {
  return (
    <div className="absolute inset-0 z-[60] bg-black/60 flex items-center justify-center p-4">
      <div className="bg-white rounded-2xl p-6 w-full max-w-sm">
        <h3 className="font-bold mb-2">{title}</h3>
        <p className="text-sm text-gray-600 mb-4">{content}</p>
        <div className="flex gap-2">
          <button onClick={onCancel} disabled={isLoading} className="flex-1 py-3 bg-gray-100 rounded-xl font-bold">취소</button>
          <button onClick={onConfirm} disabled={isLoading} className="flex-1 py-3 bg-emerald-600 text-white rounded-xl font-bold flex justify-center gap-2">
            {isLoading ? <Loader2 className="animate-spin" size={20}/> : '확인'}
          </button>
        </div>
      </div>
    </div>
  );
}