[edit_app.py](https://github.com/user-attachments/files/28625194/edit_app.py)
from flask import Flask, render_template_string, request, jsonify, session, redirect, url_for
from datetime import datetime, timedelta
import sqlite3
from functools import wraps

app = Flask(__name__)
app.secret_key = 'edit_app_secret_key'
app.config['SESSION_PERMANENT'] = True
app.config['PERMANENT_SESSION_LIFETIME'] = timedelta(days=7)

# ================= HELPER FUNCTIONS =================

def login_required(f):
    @wraps(f)
    def decorated_function(*args, **kwargs):
        if not session.get('logged_in'):
            return redirect(url_for('login'))
        return f(*args, **kwargs)
    return decorated_function

def get_db():
    conn = sqlite3.connect('attendance.db')
    conn.row_factory = sqlite3.Row
    return conn

def format_date(date_str):
    try:
        dt = datetime.strptime(date_str, '%Y-%m-%d')
        months = ['Jan', 'Feb', 'Mar', 'Apr', 'May', 'Jun', 'Jul', 'Aug', 'Sep', 'Oct', 'Nov', 'Dec']
        return f"{dt.day:02d} - {months[dt.month-1]}"
    except:
        return date_str

# ================= LOGIN PAGE =================

LOGIN_PAGE = """
<!DOCTYPE html>
<html>
<head>
    <title>Login - Edit Attendance</title>
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <style>
        * { margin: 0; padding: 0; box-sizing: border-box; }
        body {
            font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
            background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
            display: flex;
            justify-content: center;
            align-items: center;
            height: 100vh;
        }
        .login-container {
            background: white;
            border-radius: 20px;
            box-shadow: 0 20px 60px rgba(0,0,0,0.3);
            width: 400px;
            padding: 40px;
        }
        .logo { text-align: center; margin-bottom: 30px; }
        .logo h1 { color: #667eea; font-size: 28px; }
        .logo p { color: #666; font-size: 14px; }
        .input-group { margin-bottom: 20px; }
        .input-group label { display: block; margin-bottom: 8px; color: #333; font-weight: 500; }
        .input-group input {
            width: 100%;
            padding: 12px;
            border: 2px solid #e0e0e0;
            border-radius: 10px;
            font-size: 14px;
        }
        .input-group input:focus { outline: none; border-color: #667eea; }
        button {
            width: 100%;
            padding: 12px;
            background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
            color: white;
            border: none;
            border-radius: 10px;
            font-size: 16px;
            font-weight: 600;
            cursor: pointer;
        }
        .error { background: #fee; color: #c33; padding: 10px; border-radius: 8px; margin-bottom: 20px; text-align: center; }
    </style>
</head>
<body>
    <div class="login-container">
        <div class="logo">
            <h1>AL USMAN ENTERPRISE</h1>
            <p>Edit Attendance System</p>
        </div>
        {% if error %}
        <div class="error">{{ error }}</div>
        {% endif %}
        <form method="POST">
            <div class="input-group">
                <label>Access Code</label>
                <input type="password" name="access_code" placeholder="Enter access code" required>
            </div>
            <button type="submit">Login</button>
        </form>
    </div>
</body>
</html>
"""

# ================= EDIT PAGE =================

EDIT_PAGE = """
<!DOCTYPE html>
<html>
<head>
    <title>Edit Attendance</title>
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <link href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.0.0/css/all.min.css" rel="stylesheet">
    <style>
        * { margin: 0; padding: 0; box-sizing: border-box; }
        body { font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif; background: #f0f2f5; }
        .header {
            background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
            color: white;
            padding: 20px 30px;
        }
        .header h1 { font-size: 24px; margin-bottom: 5px; }
        .header p { font-size: 14px; opacity: 0.9; }
        .top-bar {
            background: white;
            padding: 15px 30px;
            display: flex;
            justify-content: space-between;
            align-items: center;
            flex-wrap: wrap;
            gap: 10px;
        }
        .btn {
            padding: 8px 20px;
            border: none;
            border-radius: 8px;
            cursor: pointer;
            font-weight: 500;
            display: inline-flex;
            align-items: center;
            gap: 8px;
        }
        .btn-primary { background: #667eea; color: white; }
        .btn-primary:hover { background: #5a67d8; transform: translateY(-2px); }
        .btn-danger { background: #e53e3e; color: white; }
        .btn-danger:hover { background: #c53030; transform: translateY(-2px); }
        .btn-light { background: #718096; color: white; }
        .btn-light:hover { background: #4a5568; transform: translateY(-2px); }
        .filter-section {
            background: white;
            padding: 20px 30px;
            margin: 20px;
            border-radius: 12px;
            display: flex;
            gap: 15px;
            flex-wrap: wrap;
            align-items: flex-end;
        }
        .filter-group { display: flex; flex-direction: column; gap: 5px; }
        .filter-group label { font-size: 12px; font-weight: 600; color: #4a5568; }
        .filter-group input, .filter-group select {
            padding: 8px 12px;
            border: 2px solid #e2e8f0;
            border-radius: 8px;
            font-size: 14px;
        }
        .table-container {
            background: white;
            margin: 20px;
            border-radius: 12px;
            overflow-x: auto;
        }
        table {
            width: 100%;
            border-collapse: collapse;
        }
        th, td {
            padding: 12px 15px;
            border: 1px solid #e2e8f0;
            text-align: left;
        }
        th { background: #f7fafc; font-weight: 600; }
        tr:hover { background: #f7fafc; }
        .sunday-row { background: #fff5f0; }
        .saturday-row { background: #e3f2fd; }
        .status-badge {
            display: inline-block;
            padding: 4px 12px;
            border-radius: 20px;
            font-size: 12px;
            font-weight: 600;
        }
        .status-present { background: #c6f6d5; color: #22543d; }
        .status-late { background: #feebc8; color: #7b341e; }
        .status-absent { background: #fed7d7; color: #742a2a; }
        .status-off { background: #e9d8fd; color: #44337a; }
        .btn-edit {
            background: #ed8936;
            color: white;
            padding: 5px 15px;
            border: none;
            border-radius: 5px;
            cursor: pointer;
            font-size: 12px;
        }
        .btn-edit:hover { background: #dd6b20; }
        
        /* Modal */
        .modal {
            display: none;
            position: fixed;
            top: 0;
            left: 0;
            width: 100%;
            height: 100%;
            background: rgba(0,0,0,0.5);
            z-index: 1000;
            justify-content: center;
            align-items: center;
        }
        .modal-content {
            background: white;
            border-radius: 20px;
            width: 450px;
            max-width: 90%;
            animation: slideIn 0.3s ease;
        }
        @keyframes slideIn {
            from { transform: translateY(-50px); opacity: 0; }
            to { transform: translateY(0); opacity: 1; }
        }
        .modal-header {
            background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
            color: white;
            padding: 20px;
            border-radius: 20px 20px 0 0;
            display: flex;
            justify-content: space-between;
        }
        .modal-body { padding: 20px; }
        .modal-footer {
            padding: 20px;
            border-top: 1px solid #e2e8f0;
            display: flex;
            justify-content: flex-end;
            gap: 10px;
        }
        .close { cursor: pointer; font-size: 24px; }
        .form-group { margin-bottom: 15px; }
        .form-group label { display: block; margin-bottom: 5px; font-weight: 600; color: #2d3748; }
        .form-group input {
            width: 100%;
            padding: 10px;
            border: 2px solid #e2e8f0;
            border-radius: 8px;
            font-size: 14px;
        }
        .form-group input:focus { outline: none; border-color: #667eea; }
        .btn-save { background: #48bb78; color: white; }
        .btn-save:hover { background: #38a169; }
        .btn-cancel { background: #718096; color: white; }
        .btn-cancel:hover { background: #4a5568; }
        
        .toast {
            position: fixed;
            bottom: 20px;
            right: 20px;
            background: #48bb78;
            color: white;
            padding: 12px 20px;
            border-radius: 8px;
            z-index: 1100;
            animation: slideInRight 0.3s ease;
        }
        @keyframes slideInRight {
            from { transform: translateX(100%); opacity: 0; }
            to { transform: translateX(0); opacity: 1; }
        }
        
        @media (max-width: 768px) {
            .filter-section { flex-direction: column; }
            th, td { padding: 8px 10px; font-size: 12px; }
        }
    </style>
</head>
<body>
    <div class="header">
        <h1><i class="fas fa-edit"></i> AL USMAN ENTERPRISE</h1>
        <p>Edit Attendance Records</p>
    </div>
    
    <div class="top-bar">
        <div class="btn-group">
            <button class="btn btn-light" onclick="logout()">
                <i class="fas fa-sign-out-alt"></i> Logout
            </button>
        </div>
    </div>
    
    <div class="filter-section">
        <div class="filter-group"><label>From Date</label><input type="date" id="from_date" min="2026-02-01"></div>
        <div class="filter-group"><label>To Date</label><input type="date" id="to_date" min="2026-02-01"></div>
        <div class="filter-group"><label>Device</label><select id="device"><option value="all">All</option><option value="2nd_floor">2nd Floor</option><option value="3rd_floor">3rd Floor</option></select></div>
        <div class="filter-group"><label>Employee</label><input type="text" id="employee_name" placeholder="Search..."></div>
        <button class="btn btn-primary" onclick="loadData()"><i class="fas fa-search"></i> Apply</button>
        <button class="btn btn-light" onclick="resetFilters()"><i class="fas fa-undo"></i> Reset</button>
    </div>
    
    <div class="table-container">
        <table id="attendance_table">
            <thead>
                <tr><th>Date</th><th>Day</th><th>Device</th><th>Employee</th><th>User ID</th><th>Check In</th><th>Check Out</th><th>Status</th><th>Action</th></tr>
            </thead>
            <tbody id="table_body"><tr><td colspan="9">Loading...</tr></tbody>
        </table>
    </div>
    
    <!-- Edit Modal -->
    <div id="editModal" class="modal">
        <div class="modal-content">
            <div class="modal-header">
                <h3><i class="fas fa-pen"></i> Edit Record</h3>
                <span class="close" onclick="closeModal()">&times;</span>
            </div>
            <div class="modal-body">
                <input type="hidden" id="edit_id">
                <div class="form-group">
                    <label>Employee</label>
                    <input type="text" id="edit_employee" readonly>
                </div>
                <div class="form-group">
                    <label>Date</label>
                    <input type="text" id="edit_date" readonly>
                </div>
                <div class="form-group">
                    <label>Device</label>
                    <input type="text" id="edit_device" readonly>
                </div>
                <div class="form-group">
                    <label>Check In (HH:MM)</label>
                    <input type="text" id="edit_checkin" placeholder="--" maxlength="5">
                </div>
                <div class="form-group">
                    <label>Check Out (HH:MM)</label>
                    <input type="text" id="edit_checkout" placeholder="--" maxlength="5">
                </div>
                <div class="form-group">
                    <label>Status (auto-calculated)</label>
                    <input type="text" id="edit_status" readonly>
                </div>
            </div>
            <div class="modal-footer">
                <button class="btn btn-cancel" onclick="closeModal()">Cancel</button>
                <button class="btn btn-save" onclick="saveEdit()">Save</button>
            </div>
        </div>
    </div>
    
    <script>
        let currentRecords = [];
        
        // Set default dates (last 30 days)
        const today = new Date();
        const thirtyDaysAgo = new Date();
        thirtyDaysAgo.setDate(today.getDate() - 30);
        const minDate = new Date('2026-02-01');
        let fromDate = thirtyDaysAgo < minDate ? minDate : thirtyDaysAgo;
        document.getElementById('from_date').value = fromDate.toISOString().split('T')[0];
        document.getElementById('to_date').value = today.toISOString().split('T')[0];
        
        loadData();
        
        async function loadData() {
            const tbody = document.getElementById('table_body');
            tbody.innerHTML = '<tr><td colspan="9">Loading...</tr>';
            
            const params = new URLSearchParams({
                from_date: document.getElementById('from_date').value,
                to_date: document.getElementById('to_date').value,
                device: document.getElementById('device').value,
                employee_name: document.getElementById('employee_name').value
            });
            
            try {
                const resp = await fetch('/get_records?' + params);
                const data = await resp.json();
                currentRecords = data.records;
                renderTable(currentRecords);
            } catch(e) {
                console.error(e);
                tbody.innerHTML = '<td><td colspan="9" style="color:red">Error loading data</tr>';
            }
        }
        
        function renderTable(records) {
            const tbody = document.getElementById('table_body');
            tbody.innerHTML = '';
            
            for (let r of records) {
                let rowClass = '';
                if (r.day === 'Sunday') rowClass = 'sunday-row';
                else if (r.day === 'Saturday') rowClass = 'saturday-row';
                
                let statusClass = '';
                if (r.status === 'Weekly Off') statusClass = 'status-off';
                else if (r.status.includes('Leave/Absent')) statusClass = 'status-absent';
                else if (r.status.includes('Present') || r.status.includes('Grace')) statusClass = 'status-present';
                else if (r.status.includes('Late')) statusClass = 'status-late';
                else statusClass = 'status-absent';
                
                let editButton = '';
                if (r.can_edit) {
                    editButton = `<button class="btn-edit" onclick="openEdit(${r.id})"><i class="fas fa-edit"></i> Edit</button>`;
                } else {
                    editButton = `<button class="btn-edit" style="background:#a0aec0; cursor:not-allowed" disabled><i class="fas fa-lock"></i> Locked</button>`;
                }
                
                tbody.innerHTML += `
                    <tr class="${rowClass}">
                        <td>${r.date}</td>
                        <td>${r.day}</td>
                        <td>${r.device}</td>
                        <td>${r.employee_name}</td>
                        <td>${r.user_id}</td>
                        <td>${r.check_in}</td>
                        <td>${r.check_out}</td>
                        <td><span class="status-badge ${statusClass}">${r.status}</span></td>
                        <td>${editButton}</td>
                    </tr>
                `;
            }
            if (records.length === 0) {
                tbody.innerHTML = '<tr><td colspan="9">No records found</tr>';
            }
        }
        
        function openEdit(id) {
            const record = currentRecords.find(r => r.id === id);
            if (!record) return;
            
            document.getElementById('edit_id').value = record.id;
            document.getElementById('edit_employee').value = record.employee_name;
            document.getElementById('edit_date').value = record.date;
            document.getElementById('edit_device').value = record.device;
            document.getElementById('edit_checkin').value = record.check_in !== '--' ? record.check_in : '';
            document.getElementById('edit_checkout').value = record.check_out !== '--' ? record.check_out : '';
            document.getElementById('edit_status').value = record.status;
            document.getElementById('editModal').style.display = 'flex';
        }
        
        function closeModal() {
            document.getElementById('editModal').style.display = 'none';
        }
        
        async function saveEdit() {
            const id = document.getElementById('edit_id').value;
            let check_in = document.getElementById('edit_checkin').value;
            let check_out = document.getElementById('edit_checkout').value;
            
            if (check_in === '') check_in = null;
            if (check_out === '') check_out = null;
            
            try {
                const resp = await fetch('/update_record', {
                    method: 'POST',
                    headers: { 'Content-Type': 'application/json' },
                    body: JSON.stringify({ id: id, check_in: check_in, check_out: check_out })
                });
                const result = await resp.json();
                
                if (result.success) {
                    showToast('Saved successfully!');
                    closeModal();
                    loadData();
                } else {
                    showToast('Error: ' + result.message, 'error');
                }
            } catch(e) {
                showToast('Error: ' + e.message, 'error');
            }
        }
        
        function showToast(message, type = 'success') {
            const toast = document.createElement('div');
            toast.className = 'toast';
            toast.innerHTML = `<i class="fas fa-${type === 'success' ? 'check-circle' : 'exclamation-circle'}"></i> ${message}`;
            document.body.appendChild(toast);
            setTimeout(() => toast.remove(), 3000);
        }
        
        function resetFilters() {
            const today = new Date();
            const thirtyDaysAgo = new Date();
            thirtyDaysAgo.setDate(today.getDate() - 30);
            const minDate = new Date('2026-02-01');
            let fromDate = thirtyDaysAgo < minDate ? minDate : thirtyDaysAgo;
            document.getElementById('from_date').value = fromDate.toISOString().split('T')[0];
            document.getElementById('to_date').value = today.toISOString().split('T')[0];
            document.getElementById('device').value = 'all';
            document.getElementById('employee_name').value = '';
            loadData();
        }
        
        function logout() {
            if (confirm('Are you sure you want to logout?')) {
                window.location.href = '/logout';
            }
        }
    </script>
</body>
</html>
"""

# ================= ROUTES =================

@app.route('/login', methods=['GET', 'POST'])
def login():
    if request.method == 'POST':
        if request.form.get('access_code') == 'alusman@123':
            session['logged_in'] = True
            session.permanent = True
            return redirect(url_for('index'))
        return render_template_string(LOGIN_PAGE, error='Invalid access code')
    return render_template_string(LOGIN_PAGE)

@app.route('/logout')
def logout():
    session.clear()
    return redirect(url_for('login'))

@app.route('/')
@login_required
def index():
    return render_template_string(EDIT_PAGE)

@app.route('/get_records')
@login_required
def get_records():
    from_date = request.args.get('from_date')
    to_date = request.args.get('to_date')
    device = request.args.get('device')
    employee = request.args.get('employee_name', '')

    conn = get_db()
    
    query = """SELECT a.id, a.user_id, a.timestamp, a.device_floor, a.punch_type, 
                      a.is_manually_edited, u.name as user_name 
               FROM attendance_logs a 
               LEFT JOIN users u ON a.user_id = u.user_id AND a.device_floor = u.device_floor 
               WHERE u.is_active = 1"""
    params = []
    
    if from_date:
        query += " AND DATE(a.timestamp) >= ?"
        params.append(from_date)
    if to_date:
        query += " AND DATE(a.timestamp) <= ?"
        params.append(to_date)
    if device and device != 'all':
        query += " AND a.device_floor = ?"
        params.append(device)
    
    query += " ORDER BY a.timestamp DESC"
    logs = conn.execute(query, params).fetchall()
    conn.close()
    
    # Group by user_id and date
    records = {}
    for log in logs:
        date = log['timestamp'].split()[0]
        key = f"{log['user_id']}_{date}"
        
        if key not in records:
            records[key] = {
                'id': log['id'],
                'date': date,
                'user_id': log['user_id'],
                'name': log['user_name'] or f"User {log['user_id']}",
                'device_floor': log['device_floor'],
                'device': '2nd Floor' if log['device_floor'] == '2nd_floor' else '3rd Floor',
                'check_in': None,
                'check_out': None,
                'is_manually_edited': log['is_manually_edited']
            }
        if log['punch_type'] == 0:
            records[key]['check_in'] = datetime.strptime(log['timestamp'], '%Y-%m-%d %H:%M:%S')
        else:
            records[key]['check_out'] = datetime.strptime(log['timestamp'], '%Y-%m-%d %H:%M:%S')
    
    # Apply employee filter
    if employee:
        records = {k: v for k, v in records.items() if employee.lower() in v['name'].lower()}
    
    # Build result
    result = []
    for r in records.values():
        day_name = datetime.strptime(r['date'], '%Y-%m-%d').strftime('%A')
        
        # Check if editable (not Sunday and not already manually edited)
        can_edit = (day_name != 'Sunday')
        
        check_in_str = r['check_in'].strftime('%H:%M') if r['check_in'] else '--'
        check_out_str = r['check_out'].strftime('%H:%M') if r['check_out'] else '--'
        
        # Calculate status
        status = calculate_status(r['check_in'], r['check_out'], day_name)
        
        result.append({
            'id': r['id'],
            'date': format_date(r['date']),
            'day': day_name,
            'device': r['device'],
            'user_id': r['user_id'],
            'employee_name': r['name'],
            'check_in': check_in_str,
            'check_out': check_out_str,
            'status': status,
            'can_edit': can_edit
        })
    
    return jsonify({'records': result})

@app.route('/update_record', methods=['POST'])
@login_required
def update_record():
    try:
        data = request.get_json()
        record_id = data.get('id')
        new_check_in = data.get('check_in')
        new_check_out = data.get('check_out')
        
        if not record_id:
            return jsonify({'success': False, 'message': 'No record ID'})
        
        conn = sqlite3.connect('attendance.db')
        cursor = conn.cursor()
        
        # Get current record
        cursor.execute('SELECT user_id, timestamp, device_floor, punch_type FROM attendance_logs WHERE id = ?', (record_id,))
        current = cursor.fetchone()
        if not current:
            conn.close()
            return jsonify({'success': False, 'message': 'Record not found'})
        
        user_id, old_timestamp, device_floor, punch_type = current
        date_part = old_timestamp.split()[0]
        day_name = datetime.strptime(date_part, '%Y-%m-%d').strftime('%A')
        
        # Cannot edit Sunday records
        if day_name == 'Sunday':
            conn.close()
            return jsonify({'success': False, 'message': 'Sunday records cannot be edited'})
        
        # Get existing check-in and check-out IDs for this day
        cursor.execute('SELECT id, punch_type FROM attendance_logs WHERE user_id = ? AND DATE(timestamp) = ? AND device_floor = ?', 
                       (user_id, date_part, device_floor))
        existing = cursor.fetchall()
        
        check_in_id = None
        check_out_id = None
        for ex in existing:
            if ex[1] == 0:
                check_in_id = ex[0]
            else:
                check_out_id = ex[0]
        
        # Update or insert check-in
        if new_check_in and str(new_check_in).strip():
            try:
                time_str = str(new_check_in).strip()
                if ':' in time_str:
                    parts = time_str.split(':')
                    h = int(parts[0]) if parts[0].isdigit() else 9
                    m = int(parts[1]) if parts[1].isdigit() else 0
                else:
                    h = 9
                    m = 0
                h = max(0, min(23, h))
                m = max(0, min(59, m))
                new_time = datetime.strptime(f"{date_part} {h:02d}:{m:02d}:00", '%Y-%m-%d %H:%M:%S')
                
                if check_in_id:
                    cursor.execute('UPDATE attendance_logs SET timestamp = ?, is_manually_edited = 1, last_edited_at = ? WHERE id = ?', 
                                   (new_time, datetime.now(), check_in_id))
                else:
                    cursor.execute('INSERT INTO attendance_logs (user_id, timestamp, device_floor, punch_type, is_manually_edited, last_edited_at) VALUES (?, ?, ?, ?, 1, ?)', 
                                   (user_id, new_time, device_floor, 0, datetime.now()))
            except Exception as e:
                print(f"Check-in error: {e}")
        
        # Update or insert check-out
        if new_check_out and str(new_check_out).strip():
            try:
                time_str = str(new_check_out).strip()
                if ':' in time_str:
                    parts = time_str.split(':')
                    h = int(parts[0]) if parts[0].isdigit() else 17
                    m = int(parts[1]) if parts[1].isdigit() else 0
                else:
                    h = 17
                    m = 0
                h = max(0, min(23, h))
                m = max(0, min(59, m))
                new_time = datetime.strptime(f"{date_part} {h:02d}:{m:02d}:00", '%Y-%m-%d %H:%M:%S')
                
                if check_out_id:
                    cursor.execute('UPDATE attendance_logs SET timestamp = ?, is_manually_edited = 1, last_edited_at = ? WHERE id = ?', 
                                   (new_time, datetime.now(), check_out_id))
                else:
                    cursor.execute('INSERT INTO attendance_logs (user_id, timestamp, device_floor, punch_type, is_manually_edited, last_edited_at) VALUES (?, ?, ?, ?, 1, ?)', 
                                   (user_id, new_time, device_floor, 1, datetime.now()))
            except Exception as e:
                print(f"Check-out error: {e}")
        
        conn.commit()
        conn.close()
        return jsonify({'success': True, 'message': 'Updated successfully'})
        
    except Exception as e:
        return jsonify({'success': False, 'message': str(e)})

def calculate_status(check_in, check_out, day_name):
    """Calculate attendance status"""
    if day_name == 'Sunday':
        return 'Weekly Off'
    
    if not check_in and not check_out:
        return 'Leave/Absent (Document your Truancy)'
    
    if not check_in or not check_out:
        return 'Missing In/Out'
    
    check_in_time = check_in.time()
    check_out_time = check_out.time()
    
    if day_name == 'Saturday':
        start = datetime.strptime('10:00', '%H:%M').time()
        grace_end = datetime.strptime('10:30', '%H:%M').time()
        end = datetime.strptime('15:30', '%H:%M').time()
        early = datetime.strptime('14:00', '%H:%M').time()
        overtime = datetime.strptime('16:00', '%H:%M').time()
    else:
        start = datetime.strptime('09:15', '%H:%M').time()
        grace_end = datetime.strptime('09:30', '%H:%M').time()
        end = datetime.strptime('17:30', '%H:%M').time()
        early = datetime.strptime('17:00', '%H:%M').time()
        overtime = datetime.strptime('18:00', '%H:%M').time()
    
    overtime_status = check_out_time > overtime
    status = ''
    
    if check_in_time <= start and check_out_time >= end:
        status = 'Present / On Time'
    elif start < check_in_time <= grace_end:
        status = 'Grace Time'
    elif check_in_time > grace_end:
        status = 'Late'
    elif check_out_time < early:
        status = 'Early Departure'
    else:
        status = 'Present'
    
    if overtime_status:
        status += ' + Overtime'
    
    return status

if __name__ == '__main__':
    app.run(host='127.0.0.1', port=5015, debug=False)

    *********************************************************************************************************************************************************************************
    [web_app.py](https://github.com/user-attachments/files/28625202/web_app.py)
from flask import Flask, render_template_string, request, jsonify, send_file, session, redirect, url_for
from datetime import datetime, timedelta
import sqlite3
from io import BytesIO, StringIO
from functools import wraps
import csv
import subprocess
import traceback
import sys
import openpyxl
from openpyxl.styles import Font, Alignment, PatternFill
from reportlab.lib.pagesizes import landscape, A4
from reportlab.platypus import SimpleDocTemplate, Table, TableStyle, Paragraph, Spacer
from reportlab.lib import colors
from reportlab.lib.styles import getSampleStyleSheet, ParagraphStyle

app = Flask(__name__)
app.secret_key = 'attendance_system_key'
app.config['SESSION_PERMANENT'] = True
app.config['PERMANENT_SESSION_LIFETIME'] = timedelta(days=7)

# ================= HELPER FUNCTIONS =================

def login_required(f):
    @wraps(f)
    def decorated_function(*args, **kwargs):
        if not session.get('logged_in'):
            return redirect(url_for('login'))
        return f(*args, **kwargs)
    return decorated_function

def get_db():
    conn = sqlite3.connect('attendance.db')
    conn.row_factory = sqlite3.Row
    return conn

def init_active_column():
    conn = sqlite3.connect('attendance.db')
    cursor = conn.cursor()
    try:
        cursor.execute("ALTER TABLE users ADD COLUMN is_active INTEGER DEFAULT 1")
        conn.commit()
    except:
        pass
    conn.close()

def format_date_web(date_str):
    try:
        dt = datetime.strptime(date_str, '%Y-%m-%d')
        return dt.strftime('%d-%b')
    except:
        return date_str

def format_date_report(date_str):
    try:
        dt = datetime.strptime(date_str, '%Y-%m-%d')
        return dt.strftime('%d-%b-%Y')
    except:
        return date_str

def get_remark_in(check_in_time, day_name):
    if not check_in_time:
        return 'MISSING IN'
    check_in = check_in_time.time()
    if day_name == 'Saturday':
        start = datetime.strptime('10:00', '%H:%M').time()
        grace_end = datetime.strptime('10:30', '%H:%M').time()
    else:
        start = datetime.strptime('09:15', '%H:%M').time()
        grace_end = datetime.strptime('09:30', '%H:%M').time()
    if check_in <= start:
        return 'ON TIME'
    elif check_in <= grace_end:
        return 'GRACE'
    else:
        return 'LATE'

def get_remark_out(check_out_time, day_name):
    if not check_out_time:
        return 'MISSING OUT'
    check_out = check_out_time.time()
    if day_name == 'Saturday':
        early = datetime.strptime('14:00', '%H:%M').time()
        overtime = datetime.strptime('16:00', '%H:%M').time()
    else:
        early = datetime.strptime('17:00', '%H:%M').time()
        overtime = datetime.strptime('18:00', '%H:%M').time()
    if check_out > overtime:
        return 'OVERTIME'
    elif check_out < early:
        return 'EARLY DEPT'
    else:
        return 'ON TIME'

def calculate_status(check_in, check_out, day_name):
    if day_name == 'Sunday':
        return 'Weekly Off'
    if not check_in and not check_out:
        return 'Leave/Absent (Document your Truancy)'
    if not check_in or not check_out:
        return 'Missing In/Out'
    check_in_time = check_in.time()
    check_out_time = check_out.time()
    if day_name == 'Saturday':
        start = datetime.strptime('10:00', '%H:%M').time()
        grace_end = datetime.strptime('10:30', '%H:%M').time()
        end = datetime.strptime('15:30', '%H:%M').time()
        early = datetime.strptime('14:00', '%H:%M').time()
        overtime = datetime.strptime('16:00', '%H:%M').time()
    else:
        start = datetime.strptime('09:15', '%H:%M').time()
        grace_end = datetime.strptime('09:30', '%H:%M').time()
        end = datetime.strptime('17:30', '%H:%M').time()
        early = datetime.strptime('17:00', '%H:%M').time()
        overtime = datetime.strptime('18:00', '%H:%M').time()
    overtime_status = check_out_time > overtime
    status = ''
    if check_in_time <= start and check_out_time >= end:
        status = 'Present / On Time'
    elif start < check_in_time <= grace_end:
        status = 'Grace Time'
    elif check_in_time > grace_end:
        status = 'Late'
    elif check_out_time < early:
        status = 'Early Departure'
    else:
        status = 'Present'
    if overtime_status:
        status += ' + Overtime'
    return status

# ================= LOGIN PAGE =================

LOGIN_PAGE = """
<!DOCTYPE html>
<html>
<head>
    <title>Login - Attendance System</title>
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <style>
        * { margin: 0; padding: 0; box-sizing: border-box; }
        body {
            font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
            background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
            display: flex;
            justify-content: center;
            align-items: center;
            height: 100vh;
        }
        .login-container {
            background: white;
            border-radius: 20px;
            box-shadow: 0 20px 60px rgba(0,0,0,0.3);
            width: 400px;
            padding: 40px;
        }
        .logo { text-align: center; margin-bottom: 30px; }
        .logo h1 { color: #667eea; font-size: 28px; }
        .logo p { color: #666; font-size: 14px; }
        .input-group { margin-bottom: 20px; }
        .input-group label { display: block; margin-bottom: 8px; color: #333; font-weight: 500; }
        .input-group input {
            width: 100%;
            padding: 12px;
            border: 2px solid #e0e0e0;
            border-radius: 10px;
            font-size: 14px;
        }
        .input-group input:focus { outline: none; border-color: #667eea; }
        button {
            width: 100%;
            padding: 12px;
            background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
            color: white;
            border: none;
            border-radius: 10px;
            font-size: 16px;
            font-weight: 600;
            cursor: pointer;
        }
        .error { background: #fee; color: #c33; padding: 10px; border-radius: 8px; margin-bottom: 20px; text-align: center; }
    </style>
</head>
<body>
    <div class="login-container">
        <div class="logo">
            <h1>AL USMAN ENTERPRISE</h1>
            <p>Attendance Management System</p>
        </div>
        {% if error %}
        <div class="error">{{ error }}</div>
        {% endif %}
        <form method="POST">
            <div class="input-group">
                <label>Access Code</label>
                <input type="password" name="access_code" placeholder="Enter access code" required>
            </div>
            <button type="submit">Login</button>
        </form>
    </div>
</body>
</html>
"""

# ================= MAIN PAGE WITH PAGINATION =================

MAIN_PAGE = """
<!DOCTYPE html>
<html>
<head>
    <title>Attendance System</title>
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <link href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.0.0/css/all.min.css" rel="stylesheet">
    <style>
        * { margin: 0; padding: 0; box-sizing: border-box; }
        body { font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif; background: #f0f2f5; }
        .header {
            background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
            color: white;
            padding: 20px 30px;
        }
        .header h1 { font-size: 24px; margin-bottom: 5px; }
        .header p { font-size: 14px; opacity: 0.9; }
        .top-bar {
            background: white;
            padding: 15px 30px;
            display: flex;
            justify-content: space-between;
            align-items: center;
            flex-wrap: wrap;
            gap: 10px;
        }
        .btn-group { display: flex; gap: 10px; flex-wrap: wrap; }
        .btn {
            padding: 8px 20px;
            border: none;
            border-radius: 8px;
            cursor: pointer;
            font-weight: 500;
            display: inline-flex;
            align-items: center;
            gap: 8px;
            transition: all 0.3s;
        }
        .btn-primary { background: #667eea; color: white; }
        .btn-primary:hover { background: #5a67d8; transform: translateY(-2px); }
        .btn-success { background: #48bb78; color: white; }
        .btn-success:hover { background: #38a169; transform: translateY(-2px); }
        .btn-secondary { background: #ed8936; color: white; }
        .btn-secondary:hover { background: #dd6b20; transform: translateY(-2px); }
        .btn-danger { background: #e53e3e; color: white; }
        .btn-danger:hover { background: #c53030; transform: translateY(-2px); }
        .btn-light { background: #718096; color: white; }
        .btn-light:hover { background: #4a5568; transform: translateY(-2px); }
        .filter-section {
            background: white;
            padding: 20px 30px;
            margin: 20px;
            border-radius: 12px;
            display: flex;
            gap: 15px;
            flex-wrap: wrap;
            align-items: flex-end;
            box-shadow: 0 2px 8px rgba(0,0,0,0.08);
        }
        .filter-group { display: flex; flex-direction: column; gap: 5px; }
        .filter-group label { font-size: 12px; font-weight: 600; color: #4a5568; }
        .filter-group input, .filter-group select {
            padding: 8px 12px;
            border: 2px solid #e2e8f0;
            border-radius: 8px;
            font-size: 14px;
        }
        .filter-group input:focus, .filter-group select:focus {
            outline: none;
            border-color: #667eea;
        }
        .table-container {
            background: white;
            margin: 20px;
            border-radius: 12px;
            overflow-x: auto;
            box-shadow: 0 2px 8px rgba(0,0,0,0.08);
        }
        table {
            width: 100%;
            border-collapse: collapse;
        }
        th, td {
            padding: 12px 10px;
            border: 1px solid #e2e8f0;
            text-align: left;
            font-size: 13px;
        }
        th { background: #f7fafc; font-weight: 600; position: sticky; top: 0; }
        tr:hover { background: #f7fafc; }
        .sunday-row { background: #fff5f0; }
        .saturday-row { background: #e3f2fd; }
        .status-badge {
            display: inline-block;
            padding: 4px 10px;
            border-radius: 20px;
            font-size: 11px;
            font-weight: 600;
        }
        .status-present { background: #c6f6d5; color: #22543d; }
        .status-late { background: #feebc8; color: #7b341e; }
        .status-absent { background: #fed7d7; color: #742a2a; }
        .status-off { background: #e9d8fd; color: #44337a; }
        
        .sidebar {
            position: fixed;
            top: 0;
            right: -450px;
            width: 450px;
            height: 100%;
            background: white;
            box-shadow: -2px 0 20px rgba(0,0,0,0.15);
            transition: right 0.3s ease;
            z-index: 1000;
            overflow-y: auto;
        }
        .sidebar.open { right: 0; }
        .sidebar-header {
            background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
            color: white;
            padding: 20px;
            display: flex;
            justify-content: space-between;
        }
        .close-sidebar {
            background: none;
            border: none;
            color: white;
            font-size: 24px;
            cursor: pointer;
        }
        .floor-section {
            margin: 20px;
            border: 1px solid #e2e8f0;
            border-radius: 10px;
        }
        .floor-title {
            background: #f7fafc;
            padding: 12px 15px;
            font-weight: 600;
            border-bottom: 1px solid #e2e8f0;
        }
        .user-item {
            padding: 12px 15px;
            display: flex;
            justify-content: space-between;
            align-items: center;
            border-bottom: 1px solid #e2e8f0;
        }
        .toggle-btn {
            padding: 5px 15px;
            border-radius: 20px;
            border: none;
            cursor: pointer;
            font-size: 12px;
            font-weight: 600;
        }
        .toggle-active { background: #48bb78; color: white; }
        .toggle-inactive { background: #e53e3e; color: white; }
        
        .error-modal {
            position: fixed;
            top: 0;
            left: 0;
            width: 100%;
            height: 100%;
            background: rgba(0,0,0,0.8);
            z-index: 3000;
            display: none;
            justify-content: center;
            align-items: center;
        }
        .error-card {
            background: white;
            border-radius: 20px;
            padding: 40px;
            text-align: center;
            min-width: 400px;
            max-width: 500px;
        }
        .error-icon { font-size: 60px; margin-bottom: 20px; color: #e53e3e; }
        .error-message { font-size: 16px; color: #2d3748; margin-bottom: 20px; line-height: 1.5; }
        .error-ok {
            background: #e53e3e;
            color: white;
            padding: 10px 30px;
            border: none;
            border-radius: 8px;
            cursor: pointer;
            font-size: 16px;
        }
        
        .fetch-overlay {
            position: fixed;
            top: 0;
            left: 0;
            width: 100%;
            height: 100%;
            background: rgba(0,0,0,0.8);
            z-index: 2000;
            display: none;
            justify-content: center;
            align-items: center;
        }
        .fetch-card {
            background: white;
            border-radius: 20px;
            padding: 40px;
            text-align: center;
            min-width: 300px;
        }
        .fetch-icon { font-size: 60px; margin-bottom: 20px; }
        .fetch-message { font-size: 18px; font-weight: 500; color: #2d3748; margin-bottom: 20px; }
        .spinner {
            border: 4px solid #f3f3f3;
            border-top: 4px solid #667eea;
            border-radius: 50%;
            width: 40px;
            height: 40px;
            animation: spin 1s linear infinite;
            margin: 0 auto;
        }
        @keyframes spin {
            0% { transform: rotate(0deg); }
            100% { transform: rotate(360deg); }
        }
        
        .pagination {
            display: flex;
            justify-content: center;
            gap: 15px;
            margin: 20px;
            padding: 10px;
        }
        
        @media (max-width: 768px) {
            .sidebar { width: 100%; right: -100%; }
            .filter-section { flex-direction: column; }
            th, td { padding: 8px 5px; font-size: 11px; }
        }
    </style>
</head>
<body>
    <div class="header">
        <h1><i class="fas fa-clock"></i> AL USMAN ENTERPRISE</h1>
        <p>Attendance Management System</p>
    </div>
    
    <div class="top-bar">
        <div class="btn-group">
            <button class="btn btn-secondary" onclick="exportReport('csv')"><i class="fas fa-file-csv"></i> Ex Csv</button>
            <button class="btn btn-secondary" onclick="exportReport('xlsx')"><i class="fas fa-file-excel"></i> Ex Xls</button>
            <button class="btn btn-secondary" onclick="exportReport('pdf')"><i class="fas fa-file-pdf"></i> Ex Pdf</button>
        </div>
        <div class="btn-group">
            <button class="btn btn-success" onclick="refreshData()"><i class="fas fa-sync-alt"></i> Refresh</button>
            <button class="btn btn-primary" onclick="openSidebar()"><i class="fas fa-users"></i> Users</button>
            <button class="btn btn-danger" onclick="logout()"><i class="fas fa-sign-out-alt"></i> Logout</button>
        </div>
    </div>
    
    <div class="filter-section">
        <div class="filter-group"><label>From Date</label><input type="date" id="from_date" min="2026-02-01"></div>
        <div class="filter-group"><label>To Date</label><input type="date" id="to_date" min="2026-02-01"></div>
        <div class="filter-group"><label>Device</label><select id="device"><option value="all">All</option><option value="2nd_floor">2nd Floor</option><option value="3rd_floor">3rd Floor</option></select></div>
        <div class="filter-group"><label>Employee</label><input type="text" id="employee_name" placeholder="Search..."></div>
        <button class="btn btn-primary" onclick="resetAndLoad()"><i class="fas fa-search"></i> Apply</button>
        <button class="btn btn-light" onclick="resetFilters()"><i class="fas fa-undo"></i> Reset</button>
    </div>
    
    <div class="table-container">
        <table id="attendance_table">
            <thead>
                <tr><th>Date</th><th>Day</th><th>Device</th><th>User ID</th><th>Name</th><th>Check In</th><th>Remark (IN)</th><th>Check Out</th><th>Remark (OUT)</th><th>Status</th></tr>
            </thead>
            <tbody id="table_body"><tr><td colspan="10">Loading...</td></tr></tbody>
        </table>
    </div>
    
    <div id="pagination" class="pagination"></div>
    
    <div id="sidebar" class="sidebar">
        <div class="sidebar-header"><h3><i class="fas fa-users"></i> Manage Users</h3><button class="close-sidebar" onclick="closeSidebar()">&times;</button></div>
        <div id="users_list"><div style="padding:20px">Loading...</div></div>
    </div>
    
    <div id="fetchOverlay" class="fetch-overlay">
        <div class="fetch-card">
            <div class="fetch-icon" id="fetchIcon">⚠️</div>
            <div class="fetch-message" id="fetchMessage">Please wait – work in progress...</div>
            <div class="spinner" id="fetchSpinner"></div>
        </div>
    </div>
    
    <div id="errorModal" class="error-modal">
        <div class="error-card">
            <div class="error-icon"><i class="fas fa-exclamation-triangle"></i></div>
            <div class="error-message" id="errorMessage">An error occurred</div>
            <button class="error-ok" onclick="closeErrorModal()">OK</button>
        </div>
    </div>
    
    <script>
        let currentPage = 1;
        let totalPages = 1;
        
        const today = new Date();
        const sevenDaysAgo = new Date();
        sevenDaysAgo.setDate(today.getDate() - 7);
        const minDate = new Date('2026-02-01');
        let fromDate = sevenDaysAgo < minDate ? minDate : sevenDaysAgo;
        document.getElementById('from_date').value = fromDate.toISOString().split('T')[0];
        document.getElementById('to_date').value = today.toISOString().split('T')[0];
        
        loadData();
        
        async function loadData() {
            const tbody = document.getElementById('table_body');
            tbody.innerHTML = ' hilabikh<td colspan="10">Loading...</td></tr>';
            
            const params = new URLSearchParams({
                from_date: document.getElementById('from_date').value,
                to_date: document.getElementById('to_date').value,
                device: document.getElementById('device').value,
                employee_name: document.getElementById('employee_name').value,
                page: currentPage
            });
            
            try {
                const resp = await fetch('/get_attendance?' + params);
                const data = await resp.json();
                totalPages = data.total_pages;
                renderTable(data.records);
                renderPagination();
            } catch(e) {
                tbody.innerHTML = '<tr><td colspan="10" style="color:red">Error loading data</td></tr>';
                showError('Error loading data: ' + e.message);
            }
        }
        
        function renderTable(records) {
            const tbody = document.getElementById('table_body');
            tbody.innerHTML = '';
            
            for (let r of records) {
                let rowClass = '';
                if (r.day === 'Sunday') rowClass = 'sunday-row';
                else if (r.day === 'Saturday') rowClass = 'saturday-row';
                
                let statusClass = '';
                if (r.status === 'Weekly Off') statusClass = 'status-off';
                else if (r.status.includes('Leave/Absent')) statusClass = 'status-absent';
                else if (r.status.includes('Present') || r.status.includes('Grace')) statusClass = 'status-present';
                else if (r.status.includes('Late')) statusClass = 'status-late';
                else statusClass = 'status-absent';
                
                tbody.innerHTML += `
                    <tr class="${rowClass}">
                        <td>${r.date}</td>
                        <td>${r.day}</td>
                        <td>${r.device}</td>
                        <td>${r.user_id}</td>
                        <td>${escapeHtml(r.employee_name)}</td>
                        <td>${r.check_in}</td>
                        <td>${r.remark_in}</td>
                        <td>${r.check_out}</td>
                        <td>${r.remark_out}</td>
                        <td><span class="status-badge ${statusClass}">${r.status}</span></td>
                    </tr>
                `;
            }
            if (records.length === 0) {
                tbody.innerHTML = '<tr><td colspan="10">No records found</td></tr>';
            }
        }
        
        function renderPagination() {
            const container = document.getElementById('pagination');
            container.innerHTML = '';
            
            if (totalPages <= 1) return;
            
            if (currentPage > 1) {
                const prevBtn = document.createElement('button');
                prevBtn.textContent = '← Previous';
                prevBtn.className = 'btn btn-light';
                prevBtn.onclick = () => { currentPage--; loadData(); };
                container.appendChild(prevBtn);
            }
            
            const pageSpan = document.createElement('span');
            pageSpan.textContent = `Page ${currentPage} of ${totalPages}`;
            pageSpan.style.cssText = 'padding: 8px 20px; background: #667eea; color: white; border-radius: 8px;';
            container.appendChild(pageSpan);
            
            if (currentPage < totalPages) {
                const nextBtn = document.createElement('button');
                nextBtn.textContent = 'Next →';
                nextBtn.className = 'btn btn-light';
                nextBtn.onclick = () => { currentPage++; loadData(); };
                container.appendChild(nextBtn);
            }
        }
        
        function resetAndLoad() {
            currentPage = 1;
            loadData();
        }
        
        function escapeHtml(text) {
            const div = document.createElement('div');
            div.textContent = text;
            return div.innerHTML;
        }
        
        function showError(message) {
            document.getElementById('errorMessage').innerHTML = message;
            document.getElementById('errorModal').style.display = 'flex';
        }
        
        function closeErrorModal() {
            document.getElementById('errorModal').style.display = 'none';
        }
        
        async function refreshData() {
            const overlay = document.getElementById('fetchOverlay');
            const icon = document.getElementById('fetchIcon');
            const message = document.getElementById('fetchMessage');
            const spinner = document.getElementById('fetchSpinner');
            
            overlay.style.display = 'flex';
            icon.innerHTML = '⚠️';
            message.innerHTML = 'Please wait – work in progress...';
            spinner.style.display = 'block';
            
            try {
                const resp = await fetch('/run_fetch', { method: 'POST' });
                const result = await resp.json();
                
                if (result.success) {
                    icon.innerHTML = '✅';
                    message.innerHTML = 'Done! Complete!';
                    spinner.style.display = 'none';
                    setTimeout(() => {
                        overlay.style.display = 'none';
                        currentPage = 1;
                        loadData();
                    }, 1500);
                } else {
                    overlay.style.display = 'none';
                    showError('Refresh failed: ' + result.message);
                }
            } catch(e) {
                overlay.style.display = 'none';
                showError('Network error: ' + e.message);
            }
        }
        
        function openSidebar() {
            document.getElementById('sidebar').classList.add('open');
            loadUsers();
        }
        
        function closeSidebar() {
            document.getElementById('sidebar').classList.remove('open');
        }
        
        async function loadUsers() {
            try {
                const resp = await fetch('/get_users');
                const users = await resp.json();
                renderUsers(users);
            } catch(e) {
                showError('Error loading users: ' + e.message);
            }
        }
        
        function renderUsers(users) {
            const container = document.getElementById('users_list');
            container.innerHTML = '';
            
            const floors = ['2nd_floor', '3rd_floor'];
            for (let floor of floors) {
                const floorUsers = users.filter(u => u.device_floor === floor);
                if (floorUsers.length > 0) {
                    const floorDiv = document.createElement('div');
                    floorDiv.className = 'floor-section';
                    floorDiv.innerHTML = `<div class="floor-title">${floor === '2nd_floor' ? '🏢 2nd Floor' : '🏢 3rd Floor'} (${floorUsers.length})</div>`;
                    
                    for (let user of floorUsers) {
                        const userDiv = document.createElement('div');
                        userDiv.className = 'user-item';
                        userDiv.innerHTML = `
                            <div><div class="user-name">${escapeHtml(user.name)}</div><div class="user-id">ID: ${user.user_id}</div></div>
                            <button class="toggle-btn ${user.is_active ? 'toggle-active' : 'toggle-inactive'}" 
                                    onclick="toggleUserStatus('${user.user_id}', '${user.device_floor}', ${user.is_active})">
                                ${user.is_active ? 'Active' : 'Inactive'}
                            </button>
                        `;
                        floorDiv.appendChild(userDiv);
                    }
                    container.appendChild(floorDiv);
                }
            }
        }
        
        async function toggleUserStatus(userId, floor, currentStatus) {
            try {
                const resp = await fetch('/toggle_user_status', {
                    method: 'POST',
                    headers: {'Content-Type': 'application/json'},
                    body: JSON.stringify({user_id: userId, device_floor: floor, is_active: !currentStatus})
                });
                const result = await resp.json();
                if (result.success) {
                    loadUsers();
                    currentPage = 1;
                    loadData();
                } else {
                    showError('Error toggling user status');
                }
            } catch(e) {
                showError('Error: ' + e.message);
            }
        }
        
        function exportReport(type) {
            const params = new URLSearchParams({
                from_date: document.getElementById('from_date').value,
                to_date: document.getElementById('to_date').value,
                device: document.getElementById('device').value,
                employee_name: document.getElementById('employee_name').value,
                format: type
            });
            window.location.href = '/export_attendance?' + params.toString();
        }
        
        function resetFilters() {
            const today = new Date();
            const sevenDaysAgo = new Date();
            sevenDaysAgo.setDate(today.getDate() - 7);
            const minDate = new Date('2026-02-01');
            let fromDate = sevenDaysAgo < minDate ? minDate : sevenDaysAgo;
            document.getElementById('from_date').value = fromDate.toISOString().split('T')[0];
            document.getElementById('to_date').value = today.toISOString().split('T')[0];
            document.getElementById('device').value = 'all';
            document.getElementById('employee_name').value = '';
            currentPage = 1;
            loadData();
        }
        
        function logout() {
            if (confirm('Are you sure you want to logout?')) {
                window.location.href = '/logout';
            }
        }
    </script>
</body>
</html>
"""

# ================= ROUTES =================

@app.route('/login', methods=['GET', 'POST'])
def login():
    if request.method == 'POST':
        if request.form.get('access_code') == 'alusman@123':
            session['logged_in'] = True
            session.permanent = True
            return redirect(url_for('index'))
        return render_template_string(LOGIN_PAGE, error='Invalid access code')
    return render_template_string(LOGIN_PAGE)

@app.route('/logout')
def logout():
    session.clear()
    return redirect(url_for('login'))

@app.route('/')
@login_required
def index():
    init_active_column()
    return render_template_string(MAIN_PAGE)

@app.route('/run_fetch', methods=['POST'])
@login_required
def run_fetch():
    try:
        result = subprocess.run([sys.executable, 'fetch_data.py'], 
                              capture_output=True, text=True, timeout=300)
        if result.returncode == 0:
            return jsonify({'success': True, 'message': 'Fetch completed'})
        else:
            error_msg = result.stderr[:500] if result.stderr else "Unknown error"
            return jsonify({'success': False, 'message': error_msg})
    except subprocess.TimeoutExpired:
        return jsonify({'success': False, 'message': 'Fetch timed out after 5 minutes'})
    except Exception as e:
        return jsonify({'success': False, 'message': str(e)})

@app.route('/get_attendance')
@login_required
def get_attendance():
    from_date = request.args.get('from_date')
    to_date = request.args.get('to_date')
    device = request.args.get('device')
    employee = request.args.get('employee_name', '')
    page = request.args.get('page', 1, type=int)
    per_page = 100

    conn = get_db()
    employees_query = "SELECT user_id, name, device_floor FROM users WHERE is_active = 1"
    employees = conn.execute(employees_query).fetchall()

    all_dates = []
    if from_date and to_date:
        start_date = datetime.strptime(from_date, '%Y-%m-%d')
        end_date = datetime.strptime(to_date, '%Y-%m-%d')
        current_date = start_date
        while current_date <= end_date:
            all_dates.append(current_date.strftime('%Y-%m-%d'))
            current_date += timedelta(days=1)

    query = """SELECT a.*, u.name as user_name 
               FROM attendance_logs a 
               LEFT JOIN users u ON a.user_id = u.user_id AND a.device_floor = u.device_floor 
               WHERE u.is_active = 1"""
    params = []
    if from_date:
        query += " AND DATE(a.timestamp) >= ?"
        params.append(from_date)
    if to_date:
        query += " AND DATE(a.timestamp) <= ?"
        params.append(to_date)
    if device and device != 'all':
        query += " AND a.device_floor = ?"
        params.append(device)
    query += " ORDER BY a.timestamp"
    logs = conn.execute(query, params).fetchall()

    existing_records = {}
    for log in logs:
        date = log['timestamp'].split()[0]
        key = f"{log['user_id']}_{date}"
        if key not in existing_records:
            existing_records[key] = {
                'date': date,
                'name': log['user_name'] or f"User {log['user_id']}",
                'user_id': log['user_id'],
                'device_floor': log['device_floor'],
                'device': '2nd Floor' if log['device_floor'] == '2nd_floor' else '3rd Floor',
                'check_in': None,
                'check_out': None
            }
        if log['punch_type'] == 0:
            existing_records[key]['check_in'] = datetime.strptime(log['timestamp'], '%Y-%m-%d %H:%M:%S')
        else:
            existing_records[key]['check_out'] = datetime.strptime(log['timestamp'], '%Y-%m-%d %H:%M:%S')
    conn.close()

    complete_records = {}
    for employee_item in employees:
        for date in all_dates:
            key = f"{employee_item['user_id']}_{date}"
            if key in existing_records:
                complete_records[key] = existing_records[key]
            else:
                complete_records[key] = {
                    'date': date,
                    'name': employee_item['name'],
                    'user_id': employee_item['user_id'],
                    'device_floor': employee_item['device_floor'],
                    'device': '2nd Floor' if employee_item['device_floor'] == '2nd_floor' else '3rd Floor',
                    'check_in': None,
                    'check_out': None
                }

    if employee:
        complete_records = {k: v for k, v in complete_records.items() if employee.lower() in v['name'].lower()}
    if device and device != 'all':
        device_filter = '2nd_floor' if device == '2nd_floor' else '3rd_floor'
        complete_records = {k: v for k, v in complete_records.items() if v['device_floor'] == device_filter}

    sorted_records = sorted(complete_records.values(), key=lambda x: (x['date'], x['name']))
    
    total_records = len(sorted_records)
    total_pages = (total_records + per_page - 1) // per_page
    start = (page - 1) * per_page
    end = start + per_page
    paginated_records = sorted_records[start:end]
    
    result = []
    for r in paginated_records:
        day_name = datetime.strptime(r['date'], '%Y-%m-%d').strftime('%A')
        status = calculate_status(r['check_in'], r['check_out'], day_name)
        remark_in = get_remark_in(r['check_in'], day_name)
        remark_out = get_remark_out(r['check_out'], day_name)
        formatted_date = format_date_web(r['date'])
        
        if day_name == 'Sunday':
            result.append({
                'date': formatted_date,
                'day': 'Sunday',
                'device': r['device'],
                'user_id': r['user_id'],
                'employee_name': r['name'],
                'check_in': '***** SUNDAY *****',
                'remark_in': '',
                'check_out': '',
                'remark_out': '',
                'status': 'Weekly Off'
            })
        else:
            if status == 'Leave/Absent (Document your Truancy)':
                check_in_str = '----'
                check_out_str = '----'
                remark_in = 'MISSING IN'
                remark_out = 'MISSING OUT'
            else:
                if r['check_in'] and r['check_out']:
                    check_in_str = r['check_in'].strftime('%H:%M')
                    check_out_str = r['check_out'].strftime('%H:%M')
                elif r['check_in'] and not r['check_out']:
                    check_in_str = r['check_in'].strftime('%H:%M')
                    check_out_str = '--'
                elif not r['check_in'] and r['check_out']:
                    check_in_str = '--'
                    check_out_str = r['check_out'].strftime('%H:%M')
                else:
                    check_in_str = '----'
                    check_out_str = '----'
            
            result.append({
                'date': formatted_date,
                'day': day_name,
                'device': r['device'],
                'user_id': r['user_id'],
                'employee_name': r['name'],
                'check_in': check_in_str,
                'remark_in': remark_in,
                'check_out': check_out_str,
                'remark_out': remark_out,
                'status': status
            })

    return jsonify({
        'records': result,
        'total_records': total_records,
        'total_pages': total_pages,
        'current_page': page,
        'per_page': per_page
    })

@app.route('/get_users')
@login_required
def get_users():
    conn = get_db()
    users = conn.execute("SELECT user_id, name, device_floor, is_active FROM users ORDER BY device_floor, name").fetchall()
    conn.close()
    return jsonify([dict(user) for user in users])

@app.route('/toggle_user_status', methods=['POST'])
@login_required
def toggle_user_status():
    data = request.get_json()
    user_id = data.get('user_id')
    device_floor = data.get('device_floor')
    is_active = data.get('is_active')
    
    conn = sqlite3.connect('attendance.db')
    cursor = conn.cursor()
    cursor.execute("UPDATE users SET is_active = ? WHERE user_id = ? AND device_floor = ?", 
                   (1 if is_active else 0, user_id, device_floor))
    conn.commit()
    conn.close()
    return jsonify({'success': True})

@app.route('/export_attendance')
@login_required
def export_attendance():
    from_date = request.args.get('from_date')
    to_date = request.args.get('to_date')
    device = request.args.get('device')
    employee = request.args.get('employee_name', '')
    export_format = request.args.get('format', 'csv')
    
    try:
        conn = get_db()
        employees_query = "SELECT user_id, name, device_floor FROM users WHERE is_active = 1"
        employees = conn.execute(employees_query).fetchall()
        
        all_dates = []
        if from_date and to_date:
            start_date = datetime.strptime(from_date, '%Y-%m-%d')
            end_date = datetime.strptime(to_date, '%Y-%m-%d')
            current_date = start_date
            while current_date <= end_date:
                all_dates.append(current_date.strftime('%Y-%m-%d'))
                current_date += timedelta(days=1)
        
        query = """SELECT a.*, u.name as user_name 
                   FROM attendance_logs a 
                   LEFT JOIN users u ON a.user_id = u.user_id AND a.device_floor = u.device_floor 
                   WHERE u.is_active = 1"""
        params = []
        if from_date:
            query += " AND DATE(a.timestamp) >= ?"
            params.append(from_date)
        if to_date:
            query += " AND DATE(a.timestamp) <= ?"
            params.append(to_date)
        if device and device != 'all':
            query += " AND a.device_floor = ?"
            params.append(device)
        query += " ORDER BY a.timestamp"
        logs = conn.execute(query, params).fetchall()
        
        existing_records = {}
        for log in logs:
            date = log['timestamp'].split()[0]
            key = f"{log['user_id']}_{date}"
            if key not in existing_records:
                existing_records[key] = {
                    'date': date,
                    'name': log['user_name'] or f"User {log['user_id']}",
                    'user_id': log['user_id'],
                    'device_floor': log['device_floor'],
                    'device': '2nd Floor' if log['device_floor'] == '2nd_floor' else '3rd Floor',
                    'check_in': None, 'check_out': None
                }
            if log['punch_type'] == 0:
                existing_records[key]['check_in'] = datetime.strptime(log['timestamp'], '%Y-%m-%d %H:%M:%S')
            else:
                existing_records[key]['check_out'] = datetime.strptime(log['timestamp'], '%Y-%m-%d %H:%M:%S')
        conn.close()
        
        complete_records = {}
        for employee_item in employees:
            for date in all_dates:
                key = f"{employee_item['user_id']}_{date}"
                if key in existing_records:
                    complete_records[key] = existing_records[key]
                else:
                    complete_records[key] = {
                        'date': date,
                        'name': employee_item['name'],
                        'user_id': employee_item['user_id'],
                        'device_floor': employee_item['device_floor'],
                        'device': '2nd Floor' if employee_item['device_floor'] == '2nd_floor' else '3rd Floor',
                        'check_in': None, 'check_out': None
                    }
        
        if employee:
            complete_records = {k: v for k, v in complete_records.items() if employee.lower() in v['name'].lower()}
        if device and device != 'all':
            device_filter = '2nd_floor' if device == '2nd_floor' else '3rd_floor'
            complete_records = {k: v for k, v in complete_records.items() if v['device_floor'] == device_filter}
        
        sorted_records = sorted(complete_records.values(), key=lambda x: (x['date'], x['name']))
        
        rows = []
        for r in sorted_records:
            day_name = datetime.strptime(r['date'], '%Y-%m-%d').strftime('%A')
            status = calculate_status(r['check_in'], r['check_out'], day_name)
            remark_in = get_remark_in(r['check_in'], day_name)
            remark_out = get_remark_out(r['check_out'], day_name)
            formatted_date = format_date_report(r['date'])
            
            if day_name == 'Sunday':
                rows.append([
                    formatted_date, 'Sunday', r['device'], r['user_id'], r['name'],
                    '***** SUNDAY *****', '', '', '', 'Weekly Off'
                ])
            else:
                if status == 'Leave/Absent (Document your Truancy)':
                    check_in_str = '----'
                    check_out_str = '----'
                    remark_in = 'MISSING IN'
                    remark_out = 'MISSING OUT'
                else:
                    if r['check_in'] and r['check_out']:
                        check_in_str = r['check_in'].strftime('%H:%M')
                        check_out_str = r['check_out'].strftime('%H:%M')
                    elif r['check_in'] and not r['check_out']:
                        check_in_str = r['check_in'].strftime('%H:%M')
                        check_out_str = '--'
                    elif not r['check_in'] and r['check_out']:
                        check_in_str = '--'
                        check_out_str = r['check_out'].strftime('%H:%M')
                    else:
                        check_in_str = '----'
                        check_out_str = '----'
                
                rows.append([
                    formatted_date, day_name, r['device'], r['user_id'], r['name'],
                    check_in_str, remark_in, check_out_str, remark_out, status
                ])
        
        headers = ['Date', 'Day', 'Device', 'User ID', 'Name', 'Check In', 'Remark (IN)', 'Check Out', 'Remark (OUT)', 'Status']
        
        if export_format == 'csv':
            output = BytesIO()
            text_output = StringIO()
            writer = csv.writer(text_output)
            writer.writerow(headers)
            writer.writerows(rows)
            output.write(text_output.getvalue().encode('utf-8'))
            output.seek(0)
            return send_file(output, mimetype='text/csv', as_attachment=True, 
                           download_name=f'attendance_report_{from_date}_to_{to_date}.csv')
        
        elif export_format == 'xlsx':
            wb = openpyxl.Workbook()
            ws = wb.active
            ws.title = "Attendance Report"
            ws.append(headers)
            for row in rows:
                ws.append(row)
            for cell in ws[1]:
                cell.font = Font(bold=True, color="FFFFFF")
                cell.fill = PatternFill(start_color="667eea", end_color="667eea", fill_type="solid")
                cell.alignment = Alignment(horizontal="center")
            for col in ws.columns:
                max_length = 0
                col_letter = col[0].column_letter
                for cell in col:
                    try:
                        if len(str(cell.value)) > max_length:
                            max_length = len(str(cell.value))
                    except:
                        pass
                adjusted_width = min(max_length + 2, 30)
                ws.column_dimensions[col_letter].width = adjusted_width
            output = BytesIO()
            wb.save(output)
            output.seek(0)
            return send_file(output, mimetype='application/vnd.openxmlformats-officedocument.spreadsheetml.sheet',
                           as_attachment=True, download_name=f'attendance_report_{from_date}_to_{to_date}.xlsx')
        
        elif export_format == 'pdf':
            output = BytesIO()
            doc = SimpleDocTemplate(output, pagesize=landscape(A4), rightMargin=20, leftMargin=20, topMargin=30, bottomMargin=30)
            elements = []
            styles = getSampleStyleSheet()
            title_style = ParagraphStyle('CustomTitle', parent=styles['Heading1'], fontSize=14, textColor=colors.HexColor('#667eea'))
            elements.append(Paragraph(f"Attendance Report ({from_date} to {to_date})", title_style))
            elements.append(Spacer(1, 20))
            table_data = [headers] + rows
            table = Table(table_data, repeatRows=1)
            table.setStyle(TableStyle([
                ('BACKGROUND', (0, 0), (-1, 0), colors.HexColor('#667eea')),
                ('TEXTCOLOR', (0, 0), (-1, 0), colors.white),
                ('ALIGN', (0, 0), (-1, -1), 'CENTER'),
                ('FONTNAME', (0, 0), (-1, 0), 'Helvetica-Bold'),
                ('FONTSIZE', (0, 0), (-1, 0), 8),
                ('BOTTOMPADDING', (0, 0), (-1, 0), 8),
                ('TOPPADDING', (0, 0), (-1, 0), 8),
                ('BACKGROUND', (0, 1), (-1, -1), colors.beige),
                ('GRID', (0, 0), (-1, -1), 0.5, colors.grey),
                ('FONTSIZE', (0, 1), (-1, -1), 7),
            ]))
            elements.append(table)
            doc.build(elements)
            output.seek(0)
            return send_file(output, mimetype='application/pdf', as_attachment=True,
                           download_name=f'attendance_report_{from_date}_to_{to_date}.pdf')
        
        return jsonify({'error': 'Invalid format'}), 400
        
    except Exception as e:
        print(f"Export error: {traceback.format_exc()}")
        return jsonify({'error': str(e)}), 500

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000, debug=False)

    *********************************************************************************************************************************************************************************
    [fetch_data.py](https://github.com/user-attachments/files/28625219/fetch_data.py)
import subprocess
import time
from datetime import datetime
from zk import ZK
import sqlite3
import sys

# ================= CONFIGURATION =================
WIFI_CONFIG = {
    '2nd_floor': {
        'ssid': '35170741',
        'password': None,
        'device_ip': '192.168.100.250',
        'device_port': 4370,
        'device_password': 0
    },
    '3rd_floor': {
        'ssid': '35170742',
        'password': '1122',
        'device_ip': '192.168.68.75',
        'device_port': 4370,
        'device_password': 1122
    }
}
# =================================================

def print_status(message, status="info"):
    """Print colored status messages without emojis"""
    if status == "success":
        print(f"[OK] {message}")
    elif status == "error":
        print(f"[ERROR] {message}")
    elif status == "warning":
        print(f"[WARN] {message}")
    elif status == "info":
        print(f"[INFO] {message}")
    else:
        print(message)

def get_current_wifi():
    try:
        result = subprocess.run(['netsh', 'wlan', 'show', 'interfaces'], 
                                capture_output=True, text=True, encoding='utf-8')
        for line in result.stdout.split('\n'):
            if 'SSID' in line and 'BSSID' not in line:
                ssid = line.split(':')[1].strip()
                if ssid and ssid != '':
                    print_status(f"Current WiFi: {ssid}", "info")
                    return ssid
        return None
    except Exception as e:
        print_status(f"Error getting current WiFi: {e}", "error")
        return None

def switch_wifi(ssid, password=None):
    try:
        current = get_current_wifi()
        if current == ssid:
            print_status(f"Already connected to {ssid}", "success")
            return True

        print_status(f"Switching from '{current}' to '{ssid}'...", "info")
        subprocess.run(['netsh', 'wlan', 'disconnect'], capture_output=True)
        time.sleep(2)
        
        cmd = f'netsh wlan connect name="{ssid}" ssid="{ssid}" interface="Wi-Fi"'
        subprocess.run(cmd, shell=True, capture_output=True, text=True)
        time.sleep(5)

        new_current = get_current_wifi()
        if new_current == ssid:
            print_status(f"Successfully connected to {ssid}", "success")
            return True
        else:
            print_status(f"Failed to connect to {ssid}", "error")
            return False
    except Exception as e:
        print_status(f"WiFi switch error: {e}", "error")
        return False

def init_database():
    conn = sqlite3.connect('attendance.db')
    cursor = conn.cursor()
    cursor.execute('''
        CREATE TABLE IF NOT EXISTS users (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            user_id TEXT NOT NULL,
            name TEXT NOT NULL,
            device_floor TEXT NOT NULL,
            last_sync TIMESTAMP,
            is_active INTEGER DEFAULT 1,
            UNIQUE(user_id, device_floor)
        )
    ''')
    cursor.execute('''
        CREATE TABLE IF NOT EXISTS attendance_logs (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            user_id TEXT NOT NULL,
            timestamp TIMESTAMP NOT NULL,
            device_floor TEXT NOT NULL,
            punch_type INTEGER,
            created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
            is_manually_edited INTEGER DEFAULT 0,
            last_edited_at TIMESTAMP,
            UNIQUE(user_id, timestamp, device_floor)
        )
    ''')
    conn.commit()
    conn.close()
    print_status("Database ready", "success")

def get_last_sync_time(floor):
    conn = sqlite3.connect('attendance.db')
    cursor = conn.cursor()
    cursor.execute('''
        SELECT MAX(timestamp) FROM attendance_logs 
        WHERE device_floor = ? AND is_manually_edited = 0
    ''', (floor,))
    result = cursor.fetchone()[0]
    conn.close()
    if result:
        last_sync = datetime.strptime(result, '%Y-%m-%d %H:%M:%S')
        print_status(f"Last sync for {floor}: {last_sync}", "info")
        return last_sync
    else:
        print_status(f"No existing data for {floor}, fetching all records", "warning")
        return datetime(2000, 1, 1)

def save_user_to_db(user_id, name, floor):
    conn = sqlite3.connect('attendance.db')
    cursor = conn.cursor()
    cursor.execute('''
        INSERT OR REPLACE INTO users (user_id, name, device_floor, last_sync)
        VALUES (?, ?, ?, ?)
    ''', (user_id, name, floor, datetime.now()))
    conn.commit()
    conn.close()

def save_attendance_to_db(user_id, timestamp, floor, punch_type):
    conn = sqlite3.connect('attendance.db')
    cursor = conn.cursor()
    try:
        cursor.execute('''
            SELECT is_manually_edited FROM attendance_logs 
            WHERE user_id = ? AND timestamp = ? AND device_floor = ?
        ''', (user_id, timestamp, floor))
        existing = cursor.fetchone()
        if existing and existing[0] == 1:
            conn.close()
            return False
        cursor.execute('''
            INSERT OR IGNORE INTO attendance_logs 
            (user_id, timestamp, device_floor, punch_type, is_manually_edited)
            VALUES (?, ?, ?, ?, 0)
        ''', (user_id, timestamp, floor, punch_type))
        conn.commit()
        return cursor.rowcount > 0
    except Exception as e:
        print_status(f"Error saving log for {user_id}: {e}", "error")
        return False
    finally:
        conn.close()

def fetch_device_data(floor):
    config = WIFI_CONFIG[floor]
    print(f"\n{'='*60}")
    print(f"[FETCHING] {floor.upper()}")
    print(f"   WiFi: {config['ssid']}")
    print(f"   Device IP: {config['device_ip']}")
    print(f"{'='*60}")

    if not switch_wifi(config['ssid'], config['password']):
        raise Exception(f"Cannot connect to WiFi '{config['ssid']}' for {floor}")

    print_status(f"Connecting to device at {config['device_ip']}...", "info")
    zk = ZK(config['device_ip'], port=config['device_port'], timeout=5, 
             password=config['device_password'], force_udp=False, ommit_ping=True)
    device_conn = zk.connect()

    if not device_conn:
        raise Exception(f"Cannot connect to device at {config['device_ip']} for {floor}")

    print_status("Connected to device successfully!", "success")

    try:
        device_conn.disable_device()
    except:
        pass

    print_status("Fetching users...", "info")
    users = device_conn.get_users()
    user_count = 0
    for user in users:
        if user.name:
            save_user_to_db(str(user.user_id), user.name, floor)
            user_count += 1
    print_status(f"Saved/Updated {user_count} users", "success")

    last_sync = get_last_sync_time(floor)
    print_status(f"Fetching logs newer than {last_sync}...", "info")
    all_logs = device_conn.get_attendance()

    new_count = 0
    skipped_edited = 0
    skipped_old = 0

    for log in all_logs:
        if log.timestamp > last_sync:
            if save_attendance_to_db(str(log.user_id), log.timestamp, floor, log.punch):
                new_count += 1
            else:
                skipped_edited += 1
        else:
            skipped_old += 1

    try:
        device_conn.enable_device()
    except:
        pass

    device_conn.disconnect()

    print(f"\n[RESULTS] for {floor}:")
    print_status(f"New records added: {new_count}", "success")
    if skipped_edited > 0:
        print_status(f"Skipped (manually edited): {skipped_edited}", "warning")
    if skipped_old > 0:
        print_status(f"Skipped (already synced): {skipped_old}", "info")
    
    return new_count, user_count

def main():
    print("\n" + "="*60)
    print("ATTENDANCE FETCHER - INCREMENTAL MODE")
    print("Manually edited records will NOT be overwritten")
    print("="*60)

    original_wifi = get_current_wifi()
    if original_wifi:
        print_status(f"Original WiFi saved: {original_wifi}", "info")
    else:
        print_status("No original WiFi detected", "warning")

    init_database()

    try:
        print("\n" + "-" * 30)
        print_status("STEP 1: Fetching 2nd Floor Data", "info")
        floor2_logs, floor2_users = fetch_device_data('2nd_floor')
        
        print("\n" + "-" * 30)
        print_status("STEP 2: Fetching 3rd Floor Data", "info")
        floor3_logs, floor3_users = fetch_device_data('3rd_floor')
        
    except Exception as e:
        print(f"\n{'='*60}")
        print_status(f"FETCH STOPPED: {e}", "error")
        print(f"{'='*60}")
        
        if original_wifi:
            print_status(f"Attempting to restore original WiFi: {original_wifi}", "info")
            switch_wifi(original_wifi)
        
        sys.exit(1)

    if original_wifi:
        print_status(f"Restoring original WiFi: {original_wifi}", "info")
        switch_wifi(original_wifi)
    else:
        print_status("No original WiFi to restore", "warning")

    conn = sqlite3.connect('attendance.db')
    cursor = conn.cursor()
    cursor.execute("SELECT COUNT(*) FROM users")
    total_users = cursor.fetchone()[0]
    cursor.execute("SELECT COUNT(*) FROM attendance_logs")
    total_logs = cursor.fetchone()[0]
    cursor.execute("SELECT COUNT(*) FROM attendance_logs WHERE is_manually_edited = 1")
    edited_count = cursor.fetchone()[0]
    conn.close()

    print("\n" + "="*60)
    print_status("FETCH COMPLETE!", "success")
    print("="*60)
    print(f"2nd Floor: {floor2_logs} new records, {floor2_users} users")
    print(f"3rd Floor: {floor3_logs} new records, {floor3_users} users")
    print(f"Total NEW records added: {floor2_logs + floor3_logs}")
    print(f"Total Users in DB: {total_users}")
    print(f"Total Attendance Logs in DB: {total_logs}")
    print(f"Manually Edited Records (protected): {edited_count}")
    print(f"Original WiFi restored: {original_wifi}")
    print("="*60)
    
    sys.exit(0)

if __name__ == '__main__':
    main()

    *******************************************************************************************************************************************************************************************************
    [fetch_data-last-sync.py](https://github.com/user-attachments/files/28625226/fetch_data-last-sync.py)
import subprocess
import time
from datetime import datetime
from zk import ZK
import sqlite3

# ================= CONFIGURATION =================
START_DATE = '2026-02-01'
END_DATE = '2026-05-31'

WIFI_CONFIG = {
    '2nd_floor': {
        'ssid': '35170741',
        'password': None,  # Saved in Windows
        'device_ip': '192.168.100.250',
        'device_port': 4370,
        'device_password': 0
    },
    '3rd_floor': {
        'ssid': '35170742',
        'password': '1122',
        'device_ip': '192.168.68.75',
        'device_port': 4370,
        'device_password': 1122
    }
}
# =================================================

def get_current_wifi():
    try:
        result = subprocess.run(['netsh', 'wlan', 'show', 'interfaces'], 
                                capture_output=True, text=True, encoding='utf-8')
        for line in result.stdout.split('\n'):
            if 'SSID' in line and 'BSSID' not in line:
                ssid = line.split(':')[1].strip()
                if ssid and ssid != '':
                    print(f"📡 Current WiFi: {ssid}")
                    return ssid
        return None
    except Exception as e:
        print(f"❌ Error getting current WiFi: {e}")
        return None

def switch_wifi(ssid, password=None):
    try:
        current = get_current_wifi()
        if current == ssid:
            print(f"✅ Already connected to {ssid}")
            return True

        print(f"🔄 Switching from '{current}' to '{ssid}'...")
        subprocess.run(['netsh', 'wlan', 'disconnect'], capture_output=True)
        time.sleep(2)
        
        cmd = f'netsh wlan connect name="{ssid}" ssid="{ssid}" interface="Wi-Fi"'
        subprocess.run(cmd, shell=True, capture_output=True, text=True)
        time.sleep(5)

        new_current = get_current_wifi()
        if new_current == ssid:
            print(f"✅ Successfully connected to {ssid}")
            return True
        else:
            print(f"❌ Failed to connect to {ssid}")
            return False
    except Exception as e:
        print(f"❌ WiFi switch error: {e}")
        return False

def init_database():
    conn = sqlite3.connect('attendance.db')
    cursor = conn.cursor()
    cursor.execute('''
        CREATE TABLE IF NOT EXISTS users (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            user_id TEXT NOT NULL,
            name TEXT NOT NULL,
            device_floor TEXT NOT NULL,
            last_sync TIMESTAMP,
            UNIQUE(user_id, device_floor)
        )
    ''')
    cursor.execute('''
        CREATE TABLE IF NOT EXISTS attendance_logs (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            user_id TEXT NOT NULL,
            timestamp TIMESTAMP NOT NULL,
            device_floor TEXT NOT NULL,
            punch_type INTEGER,
            created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
            is_manually_edited INTEGER DEFAULT 0,
            last_edited_at TIMESTAMP,
            UNIQUE(user_id, timestamp, device_floor)
        )
    ''')
    conn.commit()
    conn.close()
    print("✅ Database ready")

def clear_date_range(floor):
    """Delete existing records in date range for this floor"""
    conn = sqlite3.connect('attendance.db')
    cursor = conn.cursor()
    cursor.execute('''
        DELETE FROM attendance_logs 
        WHERE device_floor = ? AND DATE(timestamp) BETWEEN ? AND ?
    ''', (floor, START_DATE, END_DATE))
    deleted = cursor.rowcount
    conn.commit()
    conn.close()
    if deleted > 0:
        print(f"🗑️  Deleted {deleted} existing records for {floor} in date range")
    return deleted

def save_user_to_db(user_id, name, floor):
    conn = sqlite3.connect('attendance.db')
    cursor = conn.cursor()
    cursor.execute('''
        INSERT OR REPLACE INTO users (user_id, name, device_floor, last_sync)
        VALUES (?, ?, ?, ?)
    ''', (user_id, name, floor, datetime.now()))
    conn.commit()
    conn.close()

def save_attendance_to_db(user_id, timestamp, floor, punch_type):
    conn = sqlite3.connect('attendance.db')
    cursor = conn.cursor()
    try:
        cursor.execute('''
            INSERT INTO attendance_logs (user_id, timestamp, device_floor, punch_type, is_manually_edited)
            VALUES (?, ?, ?, ?, 0)
        ''', (user_id, timestamp, floor, punch_type))
        conn.commit()
        return True
    except Exception as e:
        print(f"⚠️  Error saving log: {e}")
        return False
    finally:
        conn.close()

def fetch_device_data(floor):
    config = WIFI_CONFIG[floor]
    print(f"\n{'='*60}")
    print(f"📍 FETCHING: {floor.upper()}")
    print(f"   WiFi: {config['ssid']}")
    print(f"   Device IP: {config['device_ip']}")
    print(f"   Date Range: {START_DATE} to {END_DATE}")
    print(f"{'='*60}")

    # Step 1: Connect to WiFi
    if not switch_wifi(config['ssid'], config['password']):
        raise Exception(f"❌ CRITICAL: Cannot connect to WiFi '{config['ssid']}' for {floor}")

    # Step 2: Connect to biometric device
    print(f"🔌 Connecting to device at {config['device_ip']}...")
    zk = ZK(config['device_ip'], port=config['device_port'], timeout=5, 
             password=config['device_password'], force_udp=False, ommit_ping=True)
    device_conn = zk.connect()

    if not device_conn:
        raise Exception(f"❌ CRITICAL: Cannot connect to device at {config['device_ip']} for {floor}")

    print("✅ Connected to device successfully!")

    try:
        device_conn.disable_device()
    except:
        pass

    # Step 3: Clear existing records in date range for this floor
    clear_date_range(floor)

    # Step 4: Fetch users
    print("👥 Fetching users...")
    users = device_conn.get_users()
    user_count = 0
    for user in users:
        if user.name:
            save_user_to_db(str(user.user_id), user.name, floor)
            user_count += 1
    print(f"✅ Saved {user_count} users")

    # Step 5: Fetch attendance logs
    print("📊 Fetching attendance logs...")
    all_logs = device_conn.get_attendance()
    
    new_count = 0
    out_of_range = 0
    
    for log in all_logs:
        log_date = str(log.timestamp).split()[0]
        if START_DATE <= log_date <= END_DATE:
            if save_attendance_to_db(str(log.user_id), log.timestamp, floor, log.punch):
                new_count += 1
        else:
            out_of_range += 1

    try:
        device_conn.enable_device()
    except:
        pass

    device_conn.disconnect()

    print(f"\n📈 RESULTS for {floor}:")
    print(f"   ✅ Users saved: {user_count}")
    print(f"   ✅ Records saved ({START_DATE} to {END_DATE}): {new_count}")
    print(f"   ⏭️  Records skipped (outside range): {out_of_range}")
    
    return new_count, user_count

def main():
    print("\n" + "="*60)
    print("🚀 PHASE 1: ONE-TIME FULL FETCH")
    print(f"📅 Date Range: {START_DATE} to {END_DATE}")
    print("⚠️  This will DELETE existing records in this date range")
    print("="*60)

    # Store original WiFi
    original_wifi = get_current_wifi()
    print(f"💾 Original WiFi saved: {original_wifi}")

    # Initialize database
    init_database()

    # Fetch both floors (stop on any error)
    try:
        print("\n" + "▶️" * 20)
        floor2_logs, floor2_users = fetch_device_data('2nd_floor')
        
        print("\n" + "▶️" * 20)
        floor3_logs, floor3_users = fetch_device_data('3rd_floor')
        
    except Exception as e:
        print(f"\n❌❌❌ FETCH STOPPED: {e}")
        # Try to restore original WiFi before exiting
        if original_wifi:
            print(f"\n🔄 Attempting to restore original WiFi: {original_wifi}")
            switch_wifi(original_wifi)
        return

    # Restore original WiFi
    if original_wifi:
        print(f"\n🔄 Restoring original WiFi: {original_wifi}")
        switch_wifi(original_wifi)
    else:
        print("\n⚠️  No original WiFi to restore")

    # Final statistics
    conn = sqlite3.connect('attendance.db')
    cursor = conn.cursor()
    cursor.execute("SELECT COUNT(*) FROM users")
    total_users = cursor.fetchone()[0]
    cursor.execute("SELECT COUNT(*) FROM attendance_logs")
    total_logs = cursor.fetchone()[0]
    cursor.execute("SELECT COUNT(*) FROM attendance_logs WHERE DATE(timestamp) BETWEEN ? AND ?", 
                   (START_DATE, END_DATE))
    range_logs = cursor.fetchone()[0]
    conn.close()

    print("\n" + "="*60)
    print("🎉 PHASE 1 FETCH COMPLETE!")
    print("="*60)
    print(f"📊 2nd Floor: {floor2_logs} records, {floor2_users} users")
    print(f"📊 3rd Floor: {floor3_logs} records, {floor3_users} users")
    print(f"📊 Total Users in DB: {total_users}")
    print(f"📊 Total Attendance Logs in DB: {total_logs}")
    print(f"📊 Logs in date range ({START_DATE} to {END_DATE}): {range_logs}")
    print(f"📡 Original WiFi restored: {original_wifi}")
    print("="*60)
    print("\n✅ Phase 1 complete. Ready for Phase 2 (incremental mode).")

if __name__ == '__main__':
    main()
