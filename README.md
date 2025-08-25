<!DOCTYPE html>
<html lang="es" class="font-sans">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
  <title>Recordatorio de Medicamentos</title>
  <link rel="preconnect" href="https://fonts.googleapis.com">
  <link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>
  <link href="https://fonts.googleapis.com/css2?family=Inter:wght@400;500;600;700&display=swap" rel="stylesheet">
  <script src="https://cdn.tailwindcss.com"></script>
  <style>
    body{font-family:'Inter',sans-serif;-webkit-font-smoothing:antialiased;-moz-osx-font-smoothing:grayscale;overscroll-behavior:none}
    .contrast-150{filter:contrast(1.5)}
    .slide-in-right{animation:slideIn .3s ease-out}
    @keyframes slideIn{from{transform:translateX(100%);opacity:0}to{transform:translateX(0);opacity:1}}
    .slide-out-right{animation:slideOut .3s ease-in forwards}
    @keyframes slideOut{from{transform:translateX(0);opacity:1}to{transform:translateX(100%);opacity:0}}
    .content-scroll{-ms-overflow-style:none;scrollbar-width:none;overflow-y:scroll}
    .content-scroll::-webkit-scrollbar{display:none}
    .animate-pop-in{animation:popIn .3s ease-out forwards;transform:scale(.9) translateY(20px);opacity:0}
    @keyframes popIn{to{transform:scale(1) translateY(0);opacity:1}}
    .fade-in{animation:fadeIn .5s ease-in forwards;opacity:0}
    @keyframes fadeIn{to{opacity:1}}
  </style>
</head>
<body class="bg-gray-100 text-base">
  <div id="app" class="flex flex-col h-screen overflow-hidden"></div>

  <!-- Modal de mensajes -->
  <div id="modal-message" class="fixed inset-0 z-[100] hidden flex items-center justify-center p-4 backdrop-blur-sm">
    <div class="w-full max-w-sm rounded-3xl border border-gray-200 bg-white p-6 shadow-2xl animate-pop-in">
      <div class="mb-4 text-lg font-semibold text-gray-900" id="modal-text"></div>
      <div class="flex justify-end">
        <button onclick="document.getElementById('modal-message').classList.add('hidden')" class="inline-flex items-center gap-2 rounded-2xl px-4 py-2 text-sm font-medium shadow-sm transition active:scale-[0.98] bg-black text-white hover:bg-gray-900">Cerrar</button>
      </div>
    </div>
  </div>

  <script>
  // ---------------------------- Utilidades ----------------------------
  const pad = n => String(n).padStart(2, '0');
  const fmtTime = d => `${pad(new Date(d).getHours())}:${pad(new Date(d).getMinutes())}`;
  const fmtDate = d => `${pad(new Date(d).getDate())}/${pad(new Date(d).getMonth()+1)}/${new Date(d).getFullYear()}`;
  const uid = () => Math.random().toString(36).slice(2, 9);
  const minutesSince = iso => (Date.now() - new Date(iso).getTime()) / 60000;
  const localDateKey = d => `${d.getFullYear()}-${pad(d.getMonth()+1)}-${pad(d.getDate())}`;
  const todayKey = () => localDateKey(new Date());

  // ---------------------------- Modelos y estado ----------------------------
  const ROLES = [{id:'admin',label:'Administrador'},{id:'supervisor',label:'Supervisor'},{id:'confirmador',label:'Confirmador'}];

  let state = {
    route: location.hash.replace('#','') || '/inicio',
    users: [], patient: null, schedules: [], events: [], presence: [], changeLog: [],
    currentUserRole: 'admin',
    settings: {X: 15, Y: 30, noMolestar: false, largeText: false, offline: false},
    toasts: [], toastTimeoutId: null,
    tempUser: null
  };

  // ---------------------------- Semillas demo ----------------------------
  function seeds(){
    const admin = {id: uid(), name: 'Ana', lastName: 'Gomez', role: 'admin', contact: 'ana@mail.com'};
    const luis = {id: uid(), name: 'Luis', lastName: 'Perez', role: 'confirmador', contact: 'luis@mail.com'};
    const marta = {id: uid(), name: 'Marta', lastName: 'Diaz', role: 'confirmador', contact: 'marta@mail.com'};
    const carla = {id: uid(), name: 'Carla', lastName: 'Vargas', role: 'supervisor', contact: 'carla@mail.com'};
    const users = [admin, luis, marta, carla];
    const patient = {id: uid(), name: 'Don Jorge', dob: '1947-02-11', consent: true};

    const schedules = [
      {id: uid(), patientId: patient.id, med: 'Lisinopril', dose: '10 mg', pillsPerDose: 1, frequencyType: 'diaria', time: '07:00', stock: 30, takenToday: 0},
      {id: uid(), patientId: patient.id, med: 'Metformina', dose: '850 mg', pillsPerDose: 1, frequencyType: 'cadaNhoras', nHours: 8, stock: 90, takenToday: 0},
      {id: uid(), patientId: patient.id, med: 'Atorvastatina', dose: '20 mg', pillsPerDose: 1, frequencyType: 'cadaNhoras', nHours: 12, stock: 60, takenToday: 0},
    ];

    let events = [];
    const now = new Date();
    
    schedules.forEach(s => {
      let eventTime = new Date();
      
      if (s.frequencyType === 'diaria') {
        const [h, m] = s.time.split(':').map(Number); 
        eventTime.setHours(h, m, 0, 0); 
        if (eventTime < now) eventTime.setDate(eventTime.getDate() + 1);
      } else if (s.frequencyType === 'cadaNhoras') {
        const totalMin = now.getHours() * 60 + now.getMinutes();
        const step = s.nHours * 60;
        const rem = totalMin % step;
        const delta = rem === 0 ? 0 : (step - rem);
        eventTime = new Date(now.getTime() + delta * 60000);
        eventTime.setSeconds(0, 0);
      }
      
      events.push({
        id: uid(),
        scheduleId: s.id,
        patientId: patient.id,
        medicationNombre: s.med,
        fechaHoraProgramada: eventTime.toISOString(),
        estado: 'pendiente',
        fechaHoraConfirmada: null,
        confirmadoPor: null,
        tolerance: 15,
        escalated: false,
        reminded: false,
        syncStatus: 'synced'
      });
    });
    
    return {users, patient, schedules, events};
  }

  // ---------------------------- Render raíz ----------------------------
  function render(){
    const app = document.getElementById('app');
    let html = '';
    html += `<main class="flex-grow content-scroll pb-24 md:pb-16">`;
    
    switch(state.route){
      case '/inicio': html += renderInicio(); break;
      case '/registro': html += renderRegistro(); break;
      case '/confirmar-otp': html += renderConfirmarOTP(); break;
      case '/equipo': html += renderEquipo(); break;
      case '/paciente': html += renderPaciente(); break;
      case '/medicacion': html += renderMedicacion(); break;
      case '/hoy': html += renderHoy(); break;
      case '/bitacora': html += renderBitacora(); break;
      case '/resumen': html += renderResumen(); break;
      case '/ajustes': html += renderAjustes(); break;
      default: html += renderInicio();
    }
    
    html += `</main>`;

    if (['/hoy','/medicacion','/resumen','/ajustes'].includes(state.route)) {
      const navRoutes = [
        {path: '/hoy', icon: 'Clock', label: 'Hoy'},
        {path: '/medicacion', icon: 'Calendar', label: 'Plan'},
        {path: '/resumen', icon: 'FileText', label: 'Resumen'},
        {path: '/ajustes', icon: 'Cog', label: 'Ajustes'},
      ];
      
      html += `
        <nav class="fixed bottom-0 left-0 right-0 w-full bg-white/90 backdrop-blur-sm border-t border-gray-200">
          <div class="mx-auto flex h-16 max-w-lg items-center justify-around px-2">
            ${navRoutes.map(nav => NavTab(nav.path, nav.icon, nav.label)).join('')}
          </div>
        </nav>`;
    }

    app.innerHTML = html;
    attachEventListeners();
    renderToasts();
  }

  // ---------------------------- Listeners ----------------------------
  function attachEventListeners(){
    document.querySelectorAll('[data-go]').forEach(el => {
      el.onclick = e => {
        e.preventDefault();
        navigate(el.dataset.go);
      };
    });

    const regButton = document.querySelector('[data-action="start-reg"]');
    if (regButton) regButton.onclick = () => navigate('/registro');

    const createAccountButton = document.querySelector('[data-action="create-account"]');
    if (createAccountButton) createAccountButton.onclick = () => {
      const name = document.getElementById('reg-name').value;
      const lastName = document.getElementById('reg-last-name').value;
      const contact = document.getElementById('reg-contact').value;
      const password = document.getElementById('reg-password').value;
      
      if (!name || !lastName || !contact || !password) {
        pushToast('⚠️ Todos los campos son obligatorios.', 'warn');
        return;
      }
      
      state.tempUser = {name, lastName, contact, password};
      navigate('/confirmar-otp');
    };

    const confirmOtpButton = document.querySelector('[data-action="confirm-otp"]');
    if (confirmOtpButton) confirmOtpButton.onclick = () => {
      const otp = document.getElementById('otp-code').value;
      if (otp === '1234') {
        completeRegister();
      } else {
        pushToast('❌ Código de verificación incorrecto.', 'err');
      }
    };

    const equipoAddBtn = document.querySelector('[data-action="add-user"]');
    if (equipoAddBtn) equipoAddBtn.onclick = () => {
      const contact = document.getElementById('user-contact').value;
      const role = document.getElementById('user-role').value;
      
      if (!contact) {
        pushToast('⚠️ El contacto no puede estar vacío.', 'warn');
        return;
      }
      
      addUser(contact, role);
      document.getElementById('user-contact').value = '';
    };

    const equipoDoneBtn = document.querySelector('[data-action="done-equipo"]');
    if (equipoDoneBtn) equipoDoneBtn.onclick = () => {
      if (state.users.length < 1) {
        pushToast('⚠️ Debes agregar al menos un miembro del equipo.', 'warn');
        return;
      }
      navigate('/paciente');
    };

    const patientSaveBtn = document.querySelector('[data-action="save-patient"]');
    if (patientSaveBtn) patientSaveBtn.onclick = () => {
      const name = document.getElementById('patient-name').value;
      const dob = document.getElementById('patient-dob').value;
      
      if (!name || !dob) {
        pushToast('⚠️ Nombre y fecha de nacimiento son obligatorios.', 'warn');
        return;
      }
      
      setPatient({name, dob});
    };

    const medAddBtn = document.querySelector('[data-action="add-med"]');
    if (medAddBtn) medAddBtn.onclick = () => {
      const med = document.getElementById('med-med').value;
      const dose = document.getElementById('med-dose').value || '';
      const pills = parseInt(document.getElementById('med-pills').value, 10) || 1;
      const freqType = document.querySelector('input[name="freq-type"]:checked').value;
      
      let time = null;
      let nHours = null;
      let stock = null;

      if (freqType === 'diaria') {
        time = document.getElementById('med-time').value;
        stock = parseInt(document.getElementById('med-stock').value, 10) || 30;
        
        if (!med || !time) {
          pushToast('⚠️ Complete todos los campos de la toma diaria.', 'warn');
          return;
        }
      } else if (freqType === 'cadaNhoras') {
        nHours = parseInt(document.getElementById('med-n-hours').value, 10) || 8;
        stock = parseInt(document.getElementById('med-stock').value, 10) || 30;
        
        if (!med || !nHours) {
          pushToast('⚠️ Complete todos los campos de la toma cada X horas.', 'warn');
          return;
        }
      }

      addItem({med, dose, pillsPerDose: pills, frequencyType: freqType, time, nHours, stock});
      
      // Reset form
      document.getElementById('med-med').value = '';
      document.getElementById('med-dose').value = '';
      document.getElementById('med-pills').value = '1';
      document.getElementById('med-stock').value = '30';
      document.getElementById('med-n-hours').value = '8';
      document.getElementById('med-time').value = '08:00';
    };

    document.querySelectorAll('input[name="freq-type"]').forEach(input => {
      input.onchange = e => {
        const type = e.target.value;
        document.getElementById('freq-diaria-fields').classList.toggle('hidden', type !== 'diaria');
        document.getElementById('freq-cadaNhoras-fields').classList.toggle('hidden', type !== 'cadaNhoras');
      };
    });

    const medDoneBtn = document.querySelector('[data-action="done-med"]');
    if (medDoneBtn) medDoneBtn.onclick = () => {
      if (state.schedules.length === 0) {
        pushToast('⚠️ Debe agregar al menos una toma de medicamento.', 'warn');
        return;
      }
      navigate('/hoy');
    };

    document.querySelectorAll('[data-confirm-id]').forEach(el => {
      el.onclick = () => confirmDose(el.dataset.confirmId);
    });

    const checkinBtn = document.querySelector('[data-action="checkin"]');
    if (checkinBtn) checkinBtn.onclick = () => checkinPatient();
    
    const checkoutBtn = document.querySelector('[data-action="checkout"]');
    if (checkoutBtn) checkoutBtn.onclick = () => checkoutPatient();

    document.querySelectorAll('[data-setting]').forEach(el => {
      el.onclick = () => {
        const setting = el.dataset.setting;
        const value = el.dataset.value;
        
        if (value === 'toggle') {
          setSettings({[setting]: !state.settings[setting]});
        } else {
          setSettings({[setting]: value});
        }
      };
    });
    
    document.querySelectorAll('[data-toggle]').forEach(el => {
      el.onclick = () => {
        const setting = el.dataset.toggle;
        setSettings({[setting]: !state.settings[setting]});
      };
    });

    const x = document.getElementById('settings-X');
    if (x) x.onchange = () => {
      const v = Math.max(1, parseInt(x.value || '0', 10));
      setSettings({X: v});
    };
    
    const y = document.getElementById('settings-Y');
    if (y) y.onchange = () => {
      const v = Math.max(1, parseInt(y.value || '0', 10));
      setSettings({Y: v});
    };

    document.querySelectorAll('[data-modal-text]').forEach(el => {
      el.onclick = () => {
        const txt = el.getAttribute('data-modal-text') || '';
        document.getElementById('modal-text').innerText = txt;
        document.getElementById('modal-message').classList.remove('hidden');
      };
    });
  }

  function navigate(route) {
    state.route = route;
    location.hash = '#' + route;
    render();
  }

  // ---------------------------- Toasts ----------------------------
  function pushToast(text, kind = 'info', actions = []) {
    const id = uid();
    state.toasts.push({id, text, kind, actions});
    renderToasts();
    
    clearTimeout(state.toastTimeoutId);
    state.toastTimeoutId = setTimeout(() => {
      removeToast(id);
    }, 5000);
  }

  function removeToast(id) {
    state.toasts = state.toasts.filter(t => t.id !== id);
    renderToasts();
  }

  function renderToasts() {
    let host = document.getElementById('toast-host');
    
    if (!host) {
      host = document.createElement('div');
      host.id = 'toast-host';
      host.className = 'fixed right-4 bottom-4 z-50 flex w-[22rem] max-w-[90vw] flex-col gap-2';
      document.body.appendChild(host);
    }
    
    host.innerHTML = state.toasts.map(t => `
      <div class="rounded-2xl px-3 py-2 text-sm text-white shadow-lg slide-in-right ${t.kind === 'ok' ? 'bg-emerald-700' : t.kind === 'warn' ? 'bg-amber-700' : t.kind === 'err' ? 'bg-rose-700' : 'bg-slate-800'}">
        <div class="flex items-center justify-between gap-3">
          <span>${t.text}</span>
          <button onclick="removeToast('${t.id}')" class="rounded-lg bg-white/20 px-2 py-0.5 text-xs">Cerrar</button>
        </div>
        ${t.actions?.length ? `
          <div class="mt-2 flex gap-2">
            ${t.actions.map((a, i) => `
              <button onclick="handleToastAction('${t.id}', ${i})" class="rounded-lg bg-white px-2 py-1 text-xs font-medium text-slate-900">${a.label}</button>
            `).join('')}
          </div>
        ` : ''}
      </div>
    `).join('');
  }

  function handleToastAction(toastId, index) {
    const t = state.toasts.find(x => x.id === toastId);
    if (t && t.actions[index]) {
      t.actions[index].onClick();
      removeToast(toastId);
    }
  }

  // ---------------------------- Core ----------------------------
  function logChange(tipo, payload) {
    const currentUser = state.users.find(u => u.role === state.currentUserRole) || state.users[0];
    state.changeLog.unshift({
      id: uid(),
      patientId: state.patient?.id || '',
      tipo,
      payload,
      userId: currentUser?.id,
      fechaHora: new Date().toISOString()
    });
  }

  function calculateRemaining(schedule) {
    const eventsForSchedule = state.events.filter(e => e.scheduleId === schedule.id);
    const confirmedCount = eventsForSchedule.filter(e => e.estado === 'ok').length;
    const consumed = confirmedCount * (schedule.pillsPerDose || 1);
    const remaining = Math.max(0, (schedule.stock || 0) - consumed);
    
    let tomasPorDia = 0;
    if (schedule.frequencyType === 'diaria') {
      tomasPorDia = 1;
    } else if (schedule.frequencyType === 'cadaNhoras') {
      tomasPorDia = 24 / schedule.nHours;
    }
    
    const estimatedDays = tomasPorDia > 0 ? Math.floor(remaining / (tomasPorDia * (schedule.pillsPerDose || 1))) : 0;

    return {
      remainingDoses: remaining,
      estimatedDays: Math.max(0, estimatedDays)
    };
  }

  function confirmDose(evId) {
    const ev = state.events.find(e => e.id === evId);
    if (!ev) return;
    
    if (ev.estado === 'ok') {
      pushToast('ℹ️ Esta toma ya fue confirmada.', 'info');
      return;
    }
    
    const schedule = state.schedules.find(s => s.id === ev.scheduleId);
    if (!schedule) return;
    
    const ts = new Date();
    const currentUser = state.users.find(u => u.role === state.currentUserRole) || state.users[0];
    const offline = state.settings.offline;
    
    // Actualizar el evento como confirmado
    const changed = {
      ...ev,
      estado: 'ok',
      fechaHoraConfirmada: ts.toISOString(),
      confirmadoPor: {id: currentUser?.id, name: currentUser?.name},
      confirmLockedUntil: new Date(ts.getTime() + 5 * 60000).toISOString(),
      syncStatus: offline ? 'pending' : 'synced',
      origen: offline ? 'offline' : 'online'
    };
    
    state.events = state.events.map(x => x.id === evId ? changed : x);
    
    // Verificar si está fuera de tolerancia
    const tol = schedule.tolerance || 15;
    const diff = Math.abs(minutesSince(ev.fechaHoraProgramada));
    
    if (diff > tol) {
      const motivo = prompt(`Fuera de tolerancia (±${tol} min). Ingrese motivo:`) || 'fuera de tolerancia';
      logChange('confirmacionTardía', {evId, minutos: Math.round(diff), motivo});
    }
    
    logChange('confirmacionToma', {
      evId,
      med: changed.medicationNombre,
      ts: ts.toISOString(),
      by: changed.confirmadoPor
    });
    
    // Programar siguiente evento
    const nextTime = new Date(changed.fechaHoraProgramada);
    
    if (schedule.frequencyType === 'diaria') {
      nextTime.setDate(nextTime.getDate() + 1);
    } else if (schedule.frequencyType === 'cadaNhoras') {
      nextTime.setHours(nextTime.getHours() + schedule.nHours);
    }
    
    const nextEv = {
      ...ev,
      id: uid(),
      fechaHoraProgramada: nextTime.toISOString(),
      estado: 'pendiente',
      confirmadoPor: null,
      fechaHoraConfirmada: null,
      reminded: false,
      escalated: false,
      syncStatus: 'synced'
    };
    
    state.events.push(nextEv);
    
    // Opción de deshacer
    let undone = false;
    const undo = () => {
      if (undone) return;
      undone = true;
      
      // Eliminar el evento confirmado y el próximo evento programado
      state.events = state.events.filter(x => x.id !== changed.id && x.id !== nextEv.id);
      
      pushToast('↩️ Confirmación revertida.', 'info');
      render();
    };
    
    const {remainingDoses, estimatedDays} = calculateRemaining(schedule);
    
    pushToast(
      `✅ Confirmada. Quedan ${remainingDoses} pastillas ≈ ${estimatedDays} días.`,
      'ok',
      [{label: 'Deshacer (30 s)', onClick: undo}]
    );
    
    setTimeout(undo, 30000);
    render();
  }

  function checkinPatient() {
    const currentUser = state.users.find(u => u.role === state.currentUserRole) || state.users[0];
    const data = {
      userId: currentUser.id,
      name: currentUser.name,
      time: new Date().toISOString()
    };
    
    state.presence.push(data);
    logChange('checkin', data);
    pushToast(`Estás con ${state.patient.name}.`);
    render();
  }

  function checkoutPatient() {
    const currentUser = state.users.find(u => u.role === state.currentUserRole) || state.users[0];
    state.presence = state.presence.filter(p => p.userId !== currentUser.id);
    logChange('checkout', {userId: currentUser.id});
    pushToast(`Ya no estás con ${state.patient.name}.`);
    render();
  }

  function initializeApp() {
    window.addEventListener('hashchange', () => {
      state.route = location.hash.replace('#', '') || '/inicio';
      render();
    });
    
    render();
    setupConditionalAlerts();
    setupOfflineSync();
  }

  function setupConditionalAlerts() {
    setInterval(() => {
      if (state.settings.noMolestar || state.route === '/ajustes') return;
      
      state.events = state.events.map(ev => {
        if (ev.estado === 'ok') return ev;
        
        const diff = minutesSince(ev.fechaHoraProgramada);
        let changed = {...ev};
        
        if (!ev.reminded && diff >= state.settings.X) {
          changed.reminded = true;
          pushToast(`Recordatorio: ${changed.medicationNombre} de las ${fmtTime(changed.fechaHoraProgramada)} sigue pendiente.`, 'warn');
        }
        
        if (!ev.escalated && diff >= state.settings.Y) {
          changed.escalated = true;
          pushToast(`⚠️ Aviso: ${changed.medicationNombre} sin confirmar.`, 'err');
        }
        
        return changed;
      });
    }, 10000);
  }

  function setupOfflineSync() {
    let prevOffline = state.settings.offline;
    
    setInterval(() => {
      if (prevOffline && !state.settings.offline) {
        const pending = state.events.filter(e => e.syncStatus === 'pending').length;
        
        if (pending) {
          pushToast(`Se sincronizaron ${pending} evento(s) pendientes.`, 'info');
        }
        
        state.events = state.events.map(e => 
          e.syncStatus === 'pending' ? {...e, syncStatus: 'synced', origen: 'online'} : e
        );
        
        render();
      }
      
      prevOffline = state.settings.offline;
    }, 1000);
  }

  // ---------------------------- Mutadores ----------------------------
  function completeRegister() {
    const admin = {
      id: uid(),
      name: state.tempUser.name,
      lastName: state.tempUser.lastName,
      role: 'admin',
      contact: state.tempUser.contact,
      password: state.tempUser.password
    };
    
    state.users = [admin];
    state.tempUser = null;
    state.patient = null;
    state.schedules = [];
    state.events = [];
    
    pushToast('¡Bienvenido! Tu cuenta ha sido creada. Ahora configura tu equipo.', 'ok');
    navigate('/equipo');
  }

  function addUser(contact, role) {
    state.users.push({
      id: uid(),
      name: 'Usuario ' + (state.users.length + 1),
      contact,
      role
    });
    
    render();
  }

  function setPatient(p) {
    state.patient = {
      id: uid(),
      ...p,
      consent: true
    };
    
    navigate('/medicacion');
  }

  function addItem(item) {
    const sch = {
      id: uid(),
      ...item,
      patientId: state.patient.id,
      tolerance: 15,
      takenToday: 0
    };
    
    state.schedules.push(sch);
    
    let firstEventTime = new Date();
    
    if (item.frequencyType === 'diaria') {
      const [hh, mm] = item.time.split(':').map(Number);
      firstEventTime.setHours(hh, mm, 0, 0);
      
      if (firstEventTime < new Date()) {
        firstEventTime.setDate(firstEventTime.getDate() + 1);
      }
    } else if (item.frequencyType === 'cadaNhoras') {
      const now = new Date();
      const totalMin = now.getHours() * 60 + now.getMinutes();
      const step = item.nHours * 60;
      const rem = totalMin % step;
      const delta = rem === 0 ? 0 : (step - rem);
      
      firstEventTime = new Date(now.getTime() + delta * 60000);
      firstEventTime.setSeconds(0, 0);
    }
    
    const newEvent = {
      id: uid(),
      scheduleId: sch.id,
      patientId: state.patient.id,
      medicationNombre: item.med,
      fechaHoraProgramada: firstEventTime.toISOString(),
      estado: 'pendiente',
      tolerance: item.tolerance,
      reminded: false,
      escalated: false,
      syncStatus: 'synced'
    };
    
    state.events.push(newEvent);
    render();
  }

  function setSettings(updates) {
    state.settings = {...state.settings, ...updates};
    document.documentElement.style.fontSize = state.settings.largeText ? '17px' : '15px';
    render();
  }

  // ---------------------------- UI Helpers ----------------------------
  const svgIcons = {
    Home: `<svg xmlns="http://www.w3.org/2000/svg" width="20" height="20" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round" class="lucide lucide-home"><path d="m3 9 9-7 9 7v11a2 2 0 0 1-2 2H5a2 2 0 0 1-2-2z"/><polyline points="9 22 9 12 15 12 15 22"/></svg>`,
    Users: `<svg xmlns="http://www.w3.org/2000/svg" width="20" height="20" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round" class="lucide lucide-users"><path d="M16 21v-2a4 4 0 0 0-4-4H8a4 4 0 0 0-4 4v2"/><circle cx="12" cy="7" r="4"/><path d="M22 21v-2a4 4 0 0 0-3-3.87"/><path d="M16 3.13a4 4 0 0 1 0 7.75"/></svg>`,
    Calendar: `<svg xmlns="http://www.w3.org/2000/svg" width="20" height="20" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round" class="lucide lucide-calendar"><path d="M8 2v4"/><path d="M16 2v4"/><rect width="18" height="18" x="3" y="4" rx="2"/><path d="M3 10h18"/></svg>`,
    Clock: `<svg xmlns="http://www.w3.org/2000/svg" width="20" height="20" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round" class="lucide lucide-clock"><circle cx="12" cy="12" r="10"/><polyline points="12 6 12 12 16 14"/></svg>`,
    ArrowRightLeft: `<svg xmlns="http://www.w3.org/2000/svg" width="20" height="20" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round" class="lucide lucide-arrow-right-left"><path d="M8 3L4 7l4 4"/><path d="M4 7h16"/><path d="M16 21l4-4-4-4"/><path d="M20 17H4"/></svg>`,
    Activity: `<svg xmlns="http://www.w3.org/2000/svg" width="20" height="20" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round" class="lucide lucide-activity"><polyline points="22 12 18 12 15 21 9 3 6 12 2 12"/></svg>`,
    FileText: `<svg xmlns="http://www.w3.org/2000/svg" width="20" height="20" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round" class="lucide lucide-file-text"><path d="M15 2H6a2 2 0 0 0-2 2v16a2 2 0 0 0 2 2h12a2 2 0 0 0 2-2V7Z"/><path d="M14 2v4a2 2 0 0 0 2 2h4"/><path d="M10 9H8"/><path d="M16 13H8"/><path d="M16 17H8"/></svg>`,
    Cog: `<svg xmlns="http://www.w3.org/2000/svg" width="20" height="20" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round" class="lucide lucide-cog"><path d="M12 20a8 8 0 1 0 0-16 8 8 0 0 0 0 16Z"/><path d="M12 14a2 2 0 1 0 0-4 2 2 0 0 0 0 4Z</svg>`,
    Check: `<svg xmlns="http://www.w3.org/2000/svg" width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round" class="lucide lucide-check"><polyline points="20 6 9 17 4 12"/></svg>`,
    Phone: `<svg xmlns="http://www.w3.org/2000/svg" width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round" class="lucide lucide-phone"><path d="M22 16.92v3a2 2 0 0 1-2.18 2.05L16 21.67A18 18 0 0 1 2.33 8 18 18 0 0 1 12.05 3.82L13.08 2h3a2 2 0 0 1 2 2v3a2 2 0 0 1-2 2h-1.72a2 2 0 0 0-1.63.8l-1.84 2.8a1.26 1.26 0 0 0 1.25 1.76h1.72a2 2 0 0 0 2 2v2a2 2 0 0 1 2 2h2a2 2 0 0 1 2-2"/></svg>`,
    Undo: `<svg xmlns="http://www.w3.org/2000/svg" width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round" class="lucide lucide-undo-2"><path d="M9 14 4 9l5-5"/><path d="M4 9h10a7 7 0 0 1 7 7v2"/></svg>`,
    Shield: `<svg xmlns="http://www.w3.org/2000/svg" width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round" class="lucide lucide-shield"><path d="M12 22s8-4 8-10V5l-8-3-8 3v7c0 6 8 10 8 10"/></svg>`,
    LogOut: `<svg xmlns="http://www.w3.org/2000/svg" width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round" class="lucide lucide-log-out"><path d="M9 21H5a2 2 0 0 1-2-2V5a2 2 0 0 1 2-2h4"/><polyline points="16 17 21 12 16 7"/><line x1="21" x2="9" y1="12" y2="12"/></svg>`,
    Bell: `<svg xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round" class="lucide lucide-bell"><path d="M6 8a6 6 0 0 1 12 0c0 7.6-3 9-6 9s-6-1.4-6-9"/><path d="M10.37 21a1.94 1.94 0 0 0 3.26 0"/><path d="M12 21v2"/></svg>`,
    ArrowLeft: `<svg xmlns="http://www.w3.org/2000/svg" width="20" height="20" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round" class="lucide lucide-arrow-left"><path d="m12 19-7-7 7-7"/><path d="M19 12H5"/></svg>`,
    CheckCircle: `<svg xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round" class="lucide lucide-check-circle"><path d="M22 11.08V12a10 10 0 1 1-5.93-8.83"/><path d="M22 4 12 14.01l-3-3"/></svg>`,
    XCircle: `<svg xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round" class="lucide lucide-x-circle"><circle cx="12" cy="12" r="10"/><path d="m15 9-6 6"/><path d="m9 9 6 6"/></svg>`,
    Handshake: `<svg xmlns="http://www.w3.org/2000/svg" width="20" height="20" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round" class="lucide lucide-handshake"><path d="m12 1.5 5 5c2.24 2.24 2.24 5.76 0 8L8 20.5a5 5 0 0 1-7.5-6.5l.8-.8a5.5 5.5 0 0 1 7.21-8.15l.9.9"/><path d="m18.5 12.5-5-5c-2.24-2.24-5.76-2.24-8 0L3.5 12.5a5 5 0 0 0 7.5 6.5l.8-.8a5.5 5.5 0 0 0-7.21-8.15l-.9-.9"/></svg>`,
    PersonStanding: `<svg xmlns="http://www.w3.org/2000/svg" width="20" height="20" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round" class="lucide lucide-person-standing"><circle cx="12" cy="5" r="3"/><path d="M8 21v-2a4 4 0 0 1 4-4v0a4 4 0 0 1 4 4v2"/></svg>`,
    Pill: `<svg xmlns="http://www.w3.org/2000/svg" width="20" height="20" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round" class="lucide lucide-pill"><path d="m10 12 4-4"/><path d="M18.3 5.7a2.5 2.5 0 0 0-3.5 0L6.5 14.3a2.5 2.5 0 0 0 3.5 3.5l8.3-8.3a2.5 2.5 0 0 0 0-3.5Z"/><path d="m2 22 1.5-1.5a.5.5 0 0 1 1 0L6 22l1.5-1.5a.5.5 0 0 1 1 0L10 22"/><path d="m15.5 8.5 1.5-1.5"/></svg>`
  };

  function Button({children, className = '', variant = 'primary', icon = '', onClick = null, extraAttributes = ''}) {
    const base = "inline-flex items-center justify-center gap-2 rounded-2xl px-5 py-3 text-sm font-medium shadow-sm transition active:scale-[0.98]";
    const variants = {
      primary: "bg-black text-white hover:bg-gray-800",
      ghost: "bg-gray-200 text-gray-800 hover:bg-gray-300",
      outline: "border border-gray-300 bg-white hover:bg-gray-50",
      danger: "bg-rose-600 text-white hover:bg-rose-700"
    };
    
    return `<button ${extraAttributes} class="${base} ${variants[variant]} ${className}"${onClick ? ` onclick="${onClick}"` : ''}>${svgIcons[icon] || ''}${children}</button>`;
  }

  function Card(content, className = '') {
    return `<div class="rounded-3xl border border-gray-200 bg-white p-6 shadow-md ${className}">${content}</div>`;
  }

  function Field({label, hint = '', type = 'text', content}) {
    return `<label class="flex w-full flex-col gap-1.5"><span class="text-sm font-medium text-gray-700">${label}</span>${content}${hint ? `<span class="text-xs text-gray-500">${hint}</span>` : ''}</label>`;
  }

  function Toggle({label, checked, setting}) {
    return `<div class="flex items-center justify-between rounded-2xl border border-gray-200 bg-white px-4 py-3 shadow-sm"><span class="text-sm font-medium text-gray-700">${label}</span><button data-toggle="${setting}" class="h-6 w-11 rounded-full p-0.5 transition ${checked ? 'bg-black' : 'bg-gray-300'}"><div class="h-5 w-5 rounded-full bg-white shadow transition ${checked ? 'translate-x-5' : 'translate-x-0'}" /></button></div>`;
  }

  function NavTab(to, iconName, label) {
    const isActive = state.route === to;
    return `<button data-go="${to}" class="flex flex-col items-center gap-1 transition-colors ${isActive ? 'text-black' : 'text-gray-500 hover:text-black'}">${svgIcons[iconName] || ''}<span class="text-xs font-medium">${label}</span></button>`;
  }

  function TopBar(title, subtitle = '', actions = '') {
    return `<header class="sticky top-0 z-10 w-full border-b border-gray-200 bg-white/80 backdrop-blur-sm p-4 md:p-6 shadow-sm"><div class="mx-auto max-w-lg flex items-center justify-between gap-2"><div class="flex items-center gap-2"><div class="flex h-10 w-10 items-center justify-center rounded-xl bg-black text-white">${svgIcons.Bell}</div><div><div class="text-lg font-semibold">${title}</div>${subtitle ? `<div class="text-xs text-gray-500">${subtitle}</div>` : ''}</div></div><div class="flex items-center gap-2">${state.settings.offline ? '<span class="rounded-full bg-yellow-100 px-3 py-1 text-xs font-medium text-yellow-800">Offline</span>' : ''}${actions}</div></div></header>`;
  }

  // ---------------------------- Screens ----------------------------
  function renderInicio() {
    return `<div class="flex flex-col items-center justify-center p-6 text-center"><div class="flex h-16 w-16 items-center justify-center rounded-2xl bg-black text-white shadow-lg mb-6">${svgIcons.Bell}</div><h1 class="text-3xl font-bold text-gray-900 mb-2">Bienvenido</h1><p class="text-sm text-gray-600 mb-8 max-w-xs">App de recordatorio y seguimiento de tomas de medicamentos.</p><div class="w-full max-w-sm flex flex-col gap-3">${Button({children: 'Crear cuenta', icon: 'Check', extraAttributes: 'data-action="start-reg"', className: 'w-full'})}</div></div>`;
  }

  function renderRegistro() {
    return `<div class="max-w-md mx-auto p-4 md:p-6"><h2 class="text-2xl font-bold text-gray-900 mb-6">Registro</h2><div class="grid gap-6">${Card(`<div class="mb-2 text-sm font-semibold">Crea tu cuenta</div><div class="grid gap-4">${Field({label: 'Nombre', content: `<input id="reg-name" class="w-full rounded-xl border border-gray-300 p-3 text-sm focus:ring-1 focus:ring-black focus:border-black outline-none" placeholder="Juan" value="${state.tempUser?.name || ''}" />`})}${Field({label: 'Apellido', content: `<input id="reg-last-name" class="w-full rounded-xl border border-gray-300 p-3 text-sm focus:ring-1 focus:ring-black focus:border-black outline-none" placeholder="Pérez" value="${state.tempUser?.lastName || ''}" />`})}${Field({label: 'Correo o WhatsApp', content: `<input id="reg-contact" class="w-full rounded-xl border border-gray-300 p-3 text-sm focus:ring-1 focus:ring-black focus:border-black outline-none" placeholder="ej. tu@correo.com" value="${state.tempUser?.contact || ''}" />`})}${Field({label: 'Contraseña', content: `<input id="reg-password" type="password" class="w-full rounded-xl border border-gray-300 p-3 text-sm focus:ring-1 focus:ring-black focus:border-black outline-none" placeholder="••••••••" value="${state.tempUser?.password || ''}" />`})}</div><div class="pt-4">${Button({children: 'Continuar', icon: 'Check', extraAttributes: 'data-action="create-account"', className: 'w-full'})}</div>`)} </div></div>`;
  }

  function renderConfirmarOTP() {
    return `<div class="max-w-md mx-auto p-4 md:p-6"><h2 class="text-2xl font-bold text-gray-900 mb-6">Verificación</h2><div class="grid gap-6">${Card(`<div class="mb-2 text-sm font-semibold">Ingresa el código de 4 dígitos enviado a tu ${state.tempUser?.contact || 'correo/WhatsApp'}.</div>${Field({label: 'Código de verificación', content: `<input id="otp-code" type="text" inputmode="numeric" pattern="[0-9]*" maxlength="4" class="w-full rounded-xl border border-gray-300 p-3 text-center text-lg font-bold tracking-widest focus:ring-1 focus:ring-black focus:border-black outline-none" placeholder="1234" />`})}<div class="pt-4">${Button({children: 'Confirmar', icon: 'Check', extraAttributes: 'data-action="confirm-otp"', className: 'w-full'})}</div>`)} </div></div>`;
  }

  function renderEquipo() {
    return `<div class="max-w-md mx-auto p-4 md:p-6"><h2 class="text-2xl font-bold text-gray-900 mb-6">Invitar equipo</h2><div class="grid gap-6">${Card(`<div class="mb-4 text-sm font-semibold">Agregar un miembro</div><div class="grid gap-4">${Field({label: 'Correo o WhatsApp', content: `<input id="user-contact" class="rounded-xl border border-gray-300 p-3 text-sm" placeholder="ej. correo@dominio.com" />`})}${Field({label: 'Rol', content: `<select id="user-role" class="rounded-xl border border-gray-300 p-3 text-sm">${ROLES.map(r => `<option value="${r.id}">${r.label}</option>`).join('')}</select>`})}</div><div class="mt-4 flex gap-2">${Button({children: 'Agregar', icon: 'Users', extraAttributes: 'data-action="add-user"', className: 'w-full'})}</div>`)}${Card(`<div class="mb-4 text-sm font-semibold">Mi equipo (${state.users.length})</div><ul class="divide-y text-sm">${state.users.length ? state.users.map(u => `<li class="flex items-center justify-between py-2"><div class="flex items-center gap-2"><div class="h-6 w-6 rounded-full bg-gray-200" /><span class="font-medium">${u.name}</span></div><span class="rounded-full bg-gray-100 px-2 py-1 text-xs">${ROLES.find(x => x.id === u.role)?.label}</span></li>`).join('') : '<li class="py-2 text-xs text-gray-500">Aún sin miembros…</li>'}</ul><div class="pt-4">${Button({children: 'Continuar', variant: 'primary', icon: 'ArrowRightLeft', extraAttributes: 'data-action="done-equipo"', className: 'w-full'})}</div>`)} </div></div>`;
  }

  function renderPaciente() {
    return `<div class="max-w-md mx-auto p-4 md:p-6"><h2 class="text-2xl font-bold text-gray-900 mb-6">Información del paciente</h2><div class="grid gap-6">${Card(`<div class="mb-2 text-sm font-semibold">Crear paciente</div>${Field({label: 'Nombre completo', content: `<input id="patient-name" class="rounded-xl border border-gray-300 p-3 text-sm" value="${state.patient?.name || ''}" />`})}${Field({label: 'Fecha de nacimiento', type: 'date', content: `<input id="patient-dob" type="date" class="rounded-xl border border-gray-300 p-3 text-sm" value="${state.patient?.dob || ''}" />`})}<div class="pt-4">${Button({children: 'Guardar y continuar', icon: 'Check', extraAttributes: 'data-action="save-patient"', className: 'w-full'})}</div>`)}${Card(`<div class="text-sm text-gray-600 flex items-center gap-2">${svgIcons.Shield} Consentimiento y privacidad habilitados por defecto (demo).</div>`)} </div></div>`;
  }

  function renderMedicacion() {
    return `<div class="max-w-md mx-auto p-4 md:p-6"><h2 class="text-2xl font-bold text-gray-900 mb-6">Plan de Medicación</h2><div class="grid gap-6">${Card(`<div class="mb-4 text-sm font-semibold">Añadir una toma</div><div class="grid gap-4">${Field({label: 'Medicamento', content: '<input id="med-med" class="rounded-xl border border-gray-300 p-3 text-sm" />'})}${Field({label: 'Dosis (opcional)', content: '<input id="med-dose" class="rounded-xl border border-gray-300 p-3 text-sm" placeholder="p. ej., 50 mg" />'})}${Field({label: 'Pastillas por toma', content: '<input id="med-pills" type="number" class="rounded-xl border border-gray-300 p-3 text-sm" value="1" />'})}<div class="w-full flex flex-col gap-1.5"><span class="text-sm font-medium text-gray-700">Frecuencia</span><div class="flex gap-4"><label class="flex items-center gap-2 text-sm text-gray-700"><input type="radio" name="freq-type" value="diaria" checked class="h-4 w-4 text-black focus:ring-black"> Diaria</label><label class="flex items-center gap-2 text-sm text-gray-700"><input type="radio" name="freq-type" value="cadaNhoras" class="h-4 w-4 text-black focus:ring-black"> Cada X horas</label></div></div><div id="freq-diaria-fields" class="grid gap-4">${Field({label: 'Hora fija', type: 'time', content: '<input id="med-time" type="time" class="rounded-xl border border-gray-300 p-3 text-sm" value="08:00" />'})}${Field({label: 'Pastillas totales', type: 'number', content: '<input id="med-stock" type="number" class="rounded-xl border border-gray-300 p-3 text-sm" value="30" />'})}</div><div id="freq-cadaNhoras-fields" class="grid gap-4 hidden">${Field({label: 'Cada cuántas horas?', type: 'number', content: '<input id="med-n-hours" type="number" class="rounded-xl border border-gray-300 p-3 text-sm" value="8" />'})}${Field({label: 'Pastillas totales', type: 'number', content: '<input id="med-stock" type="number" class="rounded-xl border border-gray-300 p-3 text-sm" value="30" />'})}</div></div><div class="mt-4 flex gap-2">${Button({children: 'Añadir', icon: 'Check', extraAttributes: 'data-action="add-med"', className: 'w-full'})}</div>`)}${Card(`<div class="mb-4 text-sm font-semibold">Plan actual (${state.schedules.length})</div><div class="grid gap-3">${state.schedules.length ? state.schedules.map(s => {
      const {remainingDoses, estimatedDays} = calculateRemaining(s);
      const freqText = s.frequencyType === 'diaria' ? `Diaria a las ${s.time}` : `Cada ${s.nHours} horas`;
      
      return `<div class="rounded-xl border border-gray-200 p-3"><div class="flex items-center justify-between"><div><div class="text-sm font-medium">${s.med} <span class="text-gray-500">${s.dose ? '(' + s.dose + ')' : ''}</span></div><div class="text-xs text-gray-400">${freqText}</div></div><div class="text-right"><span class="rounded-full bg-gray-100 px-2 py-1 text-xs">±${s.tolerance || 15} min</span><div class="mt-1 text-xs text-gray-500">${remainingDoses} pastillas ≈ ${estimatedDays} días</div></div></div></div>`;
    }).join('') : '<div class="text-xs text-gray-500">Aún sin tomas…</div>'}</div><div class="mt-4">${Button({children: 'Ir a Hoy', variant: 'primary', icon: 'Clock', extraAttributes: 'data-action="done-med"', className: 'w-full'})}</div>`)} </div></div>`;
  }

  function renderHoy() {
    const eventsToday = state.events.filter(e => {
      const eventDate = localDateKey(new Date(e.fechaHoraProgramada));
      return eventDate === todayKey() || e.estado === 'pendiente';
    }).sort((a, b) => a.fechaHoraProgramada.localeCompare(b.fechaHoraProgramada));
    
    const nextPending = eventsToday.find(e => e.estado !== 'ok');
    const okCount = state.events.filter(e => e.estado === 'ok').length;
    const totalCount = state.events.length;
    const adherence = totalCount > 0 ? Math.round((okCount / totalCount) * 100) : 0;
    
    const currentUser = state.users.find(u => u.role === state.currentUserRole) || state.users[0];
    const isPresent = state.presence.some(p => p.userId === currentUser?.id);
    
    const presenceBanner = `<div class="flex items-center justify-between gap-4 p-4 text-sm bg-indigo-50 text-indigo-700 rounded-2xl border border-indigo-100 mb-6"><div class="flex items-center gap-2">${svgIcons.PersonStanding}<span>Estás con ${state.patient?.name}</span></div>${Button({children: 'Salir', variant: 'outline', onClick: 'checkoutPatient()', className: 'bg-white'})}</div>`;
    
    let pageHtml = `${TopBar('Hoy', `Adherencia: ${adherence}%`)}<div class="p-4 md:p-6 grid gap-6">${state.patient && !isPresent ? Button({children: `Estás con ${state.patient.name}`, icon: 'PersonStanding', variant: 'outline', onClick: 'checkinPatient()', className: 'w-full'}) : ''}${isPresent ? presenceBanner : ''}`;

    // Mostrar próxima toma
    if (nextPending) {
      const schedule = state.schedules.find(s => s.id === nextPending.scheduleId);
      pageHtml += `${Card(`<div class="text-base font-semibold text-center mb-2">Próxima toma</div><div class="mt-4 rounded-2xl bg-gradient-to-br from-indigo-500 to-violet-600 text-white p-6 text-center shadow-lg"><div class="text-4xl font-extrabold mb-1">${fmtTime(nextPending.fechaHoraProgramada)}</div><div class="text-lg font-medium">${nextPending.medicationNombre}</div><div class="text-sm opacity-80">${schedule?.dose || ''}</div><div class="mt-6">${Button({children: 'Toma confirmada', icon: 'Check', extraAttributes: `data-confirm-id="${nextPending.id}"`, className: 'w-full bg-white text-indigo-600 hover:bg-gray-100'})}</div></div>`)}`;
    } else {
      pageHtml += `${Card(`<div class="text-base font-semibold text-center mb-2">Próxima toma</div><div class="mt-4 rounded-2xl border border-gray-200 p-6 text-center text-sm text-gray-600">No hay tomas pendientes 🎉</div>`)}`;
    }

    // Mostrar todas las tomas de hoy
    pageHtml += `${Card(`<div class="text-base font-semibold mb-3">Todas las tomas de hoy</div><div class="grid gap-3">`;
    
    if (eventsToday.length > 0) {
      eventsToday.forEach(e => {
        const schedule = state.schedules.find(s => s.id === e.scheduleId);
        const {remainingDoses, estimatedDays} = calculateRemaining(schedule);
        
        pageHtml += `<div class="rounded-xl border border-gray-200 p-3 bg-white shadow-sm"><div class="flex items-start justify-between"><div><div class="text-sm font-semibold">${fmtTime(e.fechaHoraProgramada)} · ${e.medicationNombre}</div><div class="mt-1 text-xs text-gray-400">${remainingDoses} pastillas ≈ ${estimatedDays} días</div></div><span class="rounded-full px-2 py-1 text-xs font-medium ${e.estado === 'ok' ? 'bg-emerald-100 text-emerald-700' : 'bg-orange-100 text-orange-700'}">${e.estado === 'ok' ? 'OK' : 'Pendiente'}</span></div>`;
        
        if (e.estado !== 'ok') {
          pageHtml += `<div class="mt-3">${Button({children: 'Confirmar', icon: 'Check', extraAttributes: `data-confirm-id="${e.id}"`, className: 'w-full'})}</div>`;
        } else {
          pageHtml += `<div class="mt-3 flex items-center justify-between"><div class="text-xs text-gray-500">Confirmado por ${e.confirmadoPor?.name} a las ${fmtTime(e.fechaHoraConfirmada)}</div></div>`;
        }
        
        pageHtml += `</div>`;
      });
    } else {
      pageHtml += `<div class="text-center text-sm text-gray-500 py-4">No hay tomas programadas para hoy</div>`;
    }
    
    pageHtml += `</div></div>`; // Cierre de grid gap-3 y Card
    
    pageHtml += `</div>`; // Cierre del div principal
    
    return pageHtml;
  }

  function renderBitacora() {
    let pageHtml = `${TopBar('Bitácora')}<div class="max-w-md mx-auto p-4 md:p-6 grid gap-6">${Card(`<div class="mb-4 text-base font-semibold">Historial</div><ul class="space-y-2 text-xs text-gray-600">${state.changeLog.length ? state.changeLog.map(e => {
      const user = state.users.find(u => u.id === e.userId)?.name;
      let text = '';
      
      switch(e.tipo) {
        case 'confirmacionToma':
          text = `✅ ${user} confirmó la toma de ${e.payload.med}.`;
          break;
        case 'confirmacionTardía':
          text = `⚠️ ${user} confirmó una toma ${e.payload.minutos} min tarde. Motivo: ${e.payload.motivo}.`;
          break;
        case 'deshacer':
          text = `↩️ ${user} deshizo una confirmación.`;
          break;
        case 'checkin':
          text = `🧍 ${user} hizo check-in.`;
          break;
        case 'checkout':
          text = `🏃 ${user} hizo check-out.`;
          break;
        default:
          text = `Se registró un evento: ${e.tipo}.`;
      }
      
      return `<li class="p-3 bg-gray-50 rounded-lg border border-gray-100"><strong>[${fmtDate(e.fechaHora)} ${fmtTime(e.fechaHora)}]</strong> ${text}</li>`;
    }).join('') : '<li class="p-3 text-gray-500">Sin registros aún.</li>'}</ul>`)} </div>`;
    
    return pageHtml;
  }

  function renderResumen() {
    const ok = state.events.filter(e => e.estado === 'ok').length;
    const total = state.events.length;
    const pend = total - ok;
    const adherence = total > 0 ? Math.round((ok / total) * 100) : 0;
    
    let pageHtml = `${TopBar('Resumen', 'Informes y estadísticas')}<div class="max-w-md mx-auto p-4 md:p-6 grid gap-6">${Card(`<div class="mb-2 text-base font-semibold">Resumen del día</div><div class="text-sm">Adherencia: <span class="font-bold text-lg">${adherence}%</span></div><div class="mt-4 grid grid-cols-2 gap-2 text-sm"><div class="rounded-xl bg-emerald-50 p-4 text-center"><div class="text-xs text-emerald-700 font-medium">OK</div><div class="text-xl font-bold">${ok}</div></div><div class="rounded-xl bg-orange-50 p-4 text-center"><div class="text-xs text-orange-700 font-medium">Pendiente</div><div class="text-xl font-bold">${pend}</div></div></div>`)}${Card(`<div class="mb-2 text-base font-semibold">Pastillas restantes por medicamento</div><ul class="space-y-2 text-sm text-gray-700">${state.schedules.map(s => {
      const {remainingDoses, estimatedDays} = calculateRemaining(s);
      return `<li class="p-3 rounded-lg border border-gray-100 flex items-center justify-between"><span>${s.med}</span><span class="text-xs font-medium">${remainingDoses} pastillas ≈ ${estimatedDays} días</span></li>`;
    }).join('')}</ul><div class="pt-4">${Button({children: 'Cerrar día', variant: 'ghost', extraAttributes: `data-modal-text='Cerrar día y pasar tomas pendientes a mañana (demo)'`, className: 'w-full'})}${Button({children: 'Exportar informe', variant: 'outline', icon: 'FileText', extraAttributes: `data-modal-text='Exportar PDF (demo)'`, className: 'w-full mt-2'})}</div>`)} </div>`;
    
    return pageHtml;
  }

  function renderAjustes() {
    let pageHtml = `${TopBar('Ajustes')}<div class="max-w-md mx-auto p-4 md:p-6 grid gap-6">${Card(`<div class="mb-4 text-base font-semibold">Alertas</div>${Field({label: 'Avisar si no se toma la dosis en (minutos)', hint: 'Si se confirma después de este tiempo, se registrará como toma tardía', content: `<input id="settings-X" type="number" class="rounded-xl border border-gray-300 p-3 text-sm" value="${state.settings.X}" />`})}${Field({label: 'Escalar a supervisor si no se toma en (minutos)', hint: 'Se envía una alerta a todos los supervisores', content: `<input id="settings-Y" type="number" class="rounded-xl border border-gray-300 p-3 text-sm" value="${state.settings.Y}" />`})}${Toggle({label: 'Modo No molestar', checked: state.settings.noMolestar, setting: 'noMolestar'})}`)}${Card(`<div class="mb-4 text-base font-semibold">Accesibilidad & Conectividad</div><div class="grid gap-3">${Toggle({label: 'Texto grande', checked: state.settings.largeText, setting: 'largeText'})}${Toggle({label: 'Modo offline (simulado)', checked: state.settings.offline, setting: 'offline'})}</div><div class="mt-4 text-xs text-gray-500">Al volver online, se sincronizan confirmaciones pendientes.</div>`)}${Card(`<div class="mb-4 text-base font-semibold">Privacidad y permisos</div><div class="text-sm text-gray-600">Supervisores ven resúmenes; confirmadores registran tomas; administradores editan plan.</div><div class="mt-4 grid grid-cols-2 gap-2 text-sm">${state.users.map(u => `<div class="rounded-xl border border-gray-200 p-3"><div class="font-medium">${u.name}</div><div class="text-gray-500 text-xs">${ROLES.find(x => x.id === u.role)?.label}</div></div>`).join('')}</div><div class="pt-4">${Button({children: 'Cerrar sesión', variant: 'outline', icon: 'LogOut', extraAttributes: `data-modal-text='Cerrar sesión (demo).'`})}</div>`)} </div>`;
    
    return pageHtml;
  }

  // ---------------------------- Init ----------------------------
  window.onload = function() {
    const demo = seeds();
    state.users = demo.users;
    state.patient = demo.patient;
    state.schedules = demo.schedules;
    state.events = demo.events;
    initializeApp();
  };
  </script>
</body>
</html>
