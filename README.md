# repair-attendance-system
[Uploading web_app.py…]()
from flask import Flask, render_template, request, jsonify, send_file, session, redirect, url_for
from datetime import datetime, timedelta
import sqlite3
from io import BytesIO
from functools import wraps
import csv

app = Flask(__name__)
app.secret_key = 'attendance_system_key'

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

def calculate_status(check_in, check_out, day_name):
    """Calculate attendance status based on day type"""
    if day_name == 'Sunday':
        return 'Weekly Off'
    
    if not check_in and not check_out:
        return 'Missing In/Out'
    elif not check_in or not check_out:
        return 'Missing In/Out'
    
    check_in_time = check_in.time()
    check_out_time = check_out.time()
    
    # Saturday rules
    if day_name == 'Saturday':
        start = datetime.strptime('10:00', '%H:%M').time()
        grace_end = datetime.strptime('10:30', '%H:%M').time()
        end = datetime.strptime('15:30', '%H:%M').time()
        early = datetime.strptime('14:00', '%H:%M').time()
        overtime = datetime.strptime('16:00', '%H:%M').time()
    else:  # Monday to Friday
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

@app.route('/login', methods=['GET', 'POST'])
def login():
    if request.method == 'POST':
        if request.form.get('access_code') == 'alusman@123':
            session['logged_in'] = True
            return redirect(url_for('index'))
        return render_template('login.html', error='Invalid code')
    return render_template('login.html')

@app.route('/logout')
def logout():
    session.pop('logged_in', None)
    return redirect(url_for('login'))

@app.route('/')
@login_required
def index():
    return render_template('index.html')

@app.route('/get_attendance')
@login_required
def get_attendance():
    from_date = request.args.get('from_date')
    to_date = request.args.get('to_date')
    device = request.args.get('device')
    employee = request.args.get('employee_name', '')
    
    conn = get_db()
    
    # Get all employees
    employees_query = "SELECT user_id, name, device_floor FROM users"
    employees = conn.execute(employees_query).fetchall()
    
    # Generate all dates between from_date and to_date
    all_dates = []
    if from_date and to_date:
        start_date = datetime.strptime(from_date, '%Y-%m-%d')
        end_date = datetime.strptime(to_date, '%Y-%m-%d')
        current_date = start_date
        while current_date <= end_date:
            all_dates.append(current_date.strftime('%Y-%m-%d'))
            current_date += timedelta(days=1)
    
    # Get existing attendance logs - FIXED: Remove hardcoded date filter
    query = "SELECT a.*, u.name as user_name FROM attendance_logs a LEFT JOIN users u ON a.user_id = u.user_id AND a.device_floor = u.device_floor WHERE 1=1"
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
    
    # Create dictionary of existing records with proper check_in/check_out
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
            # This is a check-in
            existing_records[key]['check_in'] = datetime.strptime(log['timestamp'], '%Y-%m-%d %H:%M:%S')
        else:
            # This is a check-out
            existing_records[key]['check_out'] = datetime.strptime(log['timestamp'], '%Y-%m-%d %H:%M:%S')
    
    conn.close()
    
    # Build complete grid: all employees x all dates
    complete_records = {}
    for employee_item in employees:
        for date in all_dates:
            key = f"{employee_item['user_id']}_{date}"
            if key in existing_records:
                # Use existing record with actual check_in/check_out times
                complete_records[key] = existing_records[key]
            else:
                # Create missing record with no check_in/check_out
                complete_records[key] = {
                    'date': date,
                    'name': employee_item['name'],
                    'user_id': employee_item['user_id'],
                    'device_floor': employee_item['device_floor'],
                    'device': '2nd Floor' if employee_item['device_floor'] == '2nd_floor' else '3rd Floor',
                    'check_in': None,
                    'check_out': None
                }
    
    # Apply employee filter
    if employee:
        complete_records = {k:v for k,v in complete_records.items() if employee.lower() in v['name'].lower()}
    
    # Apply device filter (for missing records, use their assigned device)
    if device and device != 'all':
        device_filter = '2nd_floor' if device == '2nd_floor' else '3rd_floor'
        complete_records = {k:v for k,v in complete_records.items() if v['device_floor'] == device_filter}
    
    # Sort records by date and name
    sorted_records = sorted(complete_records.values(), key=lambda x: (x['date'], x['name']))
    
    result = []
    present = late = absent = 0
    
    for r in sorted_records:
        day_name = datetime.strptime(r['date'], '%Y-%m-%d').strftime('%A')
        status = calculate_status(r['check_in'], r['check_out'], day_name)
        
        # Count logic
        if day_name == 'Sunday':
            # Sunday is not counted as absent
            pass
        elif status == 'Missing In/Out':
            absent += 1
        elif 'Late' in status:
            late += 1
            present += 1
        elif status in ['Present / On Time', 'Grace Time', 'Present'] or 'Overtime' in status:
            present += 1
        elif status == 'Early Departure':
            present += 1
        
        # Format display values
        formatted_date = format_date(r['date'])
        
        if day_name == 'Sunday':
            # Sunday: merged Check In/Out column
            result.append({
                'date': formatted_date,
                'day': 'Sunday',
                'device': r['device'],
                'employee_name': r['name'],
                'check_in': '***** SUNDAY *****',
                'check_out': '',
                'status': 'Weekly Off',
                'original_day': day_name,
                'is_sunday': True
            })
        else:
            # Normal days - SHOW ACTUAL TIMES IF THEY EXIST
            if r['check_in'] and r['check_out']:
                # Both check-in and check-out exist - show actual times
                check_in_str = r['check_in'].strftime('%H:%M')
                check_out_str = r['check_out'].strftime('%H:%M')
            elif r['check_in'] and not r['check_out']:
                # Only check-in exists
                check_in_str = r['check_in'].strftime('%H:%M')
                check_out_str = '--'
            elif not r['check_in'] and r['check_out']:
                # Only check-out exists
                check_in_str = '--'
                check_out_str = r['check_out'].strftime('%H:%M')
            else:
                # No check-in or check-out
                check_in_str = '--'
                check_out_str = '--'
            
            result.append({
                'date': formatted_date,
                'day': day_name,
                'device': r['device'],
                'employee_name': r['name'],
                'check_in': check_in_str,
                'check_out': check_out_str,
                'status': status,
                'original_day': day_name,
                'is_sunday': False
            })
    
    total_records = len(result)
    
    return jsonify({
        'records': result, 
        'summary': {'total': total_records, 'present': present, 'late': late, 'absent': absent}
    })

@app.route('/export_attendance')
@login_required
def export_attendance():
    from_date = request.args.get('from_date')
    to_date = request.args.get('to_date')
    device = request.args.get('device')
    employee = request.args.get('employee_name', '')
    
    conn = get_db()
    
    # Get all employees
    employees_query = "SELECT user_id, name, device_floor FROM users"
    employees = conn.execute(employees_query).fetchall()
    
    # Generate all dates between from_date and to_date
    all_dates = []
    if from_date and to_date:
        start_date = datetime.strptime(from_date, '%Y-%m-%d')
        end_date = datetime.strptime(to_date, '%Y-%m-%d')
        current_date = start_date
        while current_date <= end_date:
            all_dates.append(current_date.strftime('%Y-%m-%d'))
            current_date += timedelta(days=1)
    
    # Get existing attendance logs
    query = "SELECT a.*, u.name as user_name FROM attendance_logs a LEFT JOIN users u ON a.user_id = u.user_id AND a.device_floor = u.device_floor WHERE 1=1"
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
    
    # Create dictionary of existing records
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
    
    # Build complete grid: all employees x all dates
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
    
    # Apply employee filter
    if employee:
        complete_records = {k:v for k,v in complete_records.items() if employee.lower() in v['name'].lower()}
    
    # Apply device filter
    if device and device != 'all':
        device_filter = '2nd_floor' if device == '2nd_floor' else '3rd_floor'
        complete_records = {k:v for k,v in complete_records.items() if v['device_floor'] == device_filter}
    
    # Sort records by date and name
    sorted_records = sorted(complete_records.values(), key=lambda x: (x['date'], x['name']))
    
    # Create CSV
    output = BytesIO()
    writer = csv.writer(output)
    
    # Write headers
    writer.writerow(['Date', 'Day', 'Device', 'Employee Name', 'Check In', 'Check Out', 'Status'])
    
    # Write data rows
    for r in sorted_records:
        day_name = datetime.strptime(r['date'], '%Y-%m-%d').strftime('%A')
        formatted_date = format_date(r['date'])
        status = calculate_status(r['check_in'], r['check_out'], day_name)
        
        if day_name == 'Sunday':
            writer.writerow([
                formatted_date,
                'Sunday',
                r['device'],
                r['name'],
                '***** SUNDAY *****',
                '',
                'Weekly Off'
            ])
        else:
            # Show actual times if they exist
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
                check_in_str = '--'
                check_out_str = '--'
            
            writer.writerow([
                formatted_date,
                day_name,
                r['device'],
                r['name'],
                check_in_str,
                check_out_str,
                status
            ])
    
    output.seek(0)
    return send_file(
        output,
        mimetype='text/csv',
        as_attachment=True,
        download_name=f'attendance_report_{from_date}_to_{to_date}.csv'
    )

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000, debug=False)

   ********************************************************************************************************************************************************************************************************************
   [fetch_data.py](https://github.com/user-attachments/files/28502566/fetch_data.py)
import subprocess
import time
from datetime import datetime
from zk import ZK
import sqlite3

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

def get_current_wifi():
    try:
        result = subprocess.run(['netsh', 'wlan', 'show', 'interfaces'], 
                               capture_output=True, text=True, encoding='utf-8')
        for line in result.stdout.split('\n'):
            if 'SSID' in line and 'BSSID' not in line:
                ssid = line.split(':')[1].strip()
                if ssid and ssid != '':
                    print(f"Currently connected to: {ssid}")
                    return ssid
        return None
    except Exception as e:
        print(f"Error getting current WiFi: {e}")
        return None

def switch_wifi(ssid, password=None):
    try:
        current = get_current_wifi()
        if current == ssid:
            print(f"Already connected to {ssid}")
            return True
        
        print(f"Switching from '{current}' to '{ssid}'...")
        subprocess.run(['netsh', 'wlan', 'disconnect'], capture_output=True)
        time.sleep(2)
        cmd = f'netsh wlan connect name="{ssid}" ssid="{ssid}" interface="Wi-Fi"'
        subprocess.run(cmd, shell=True, capture_output=True, text=True)
        time.sleep(5)
        
        new_current = get_current_wifi()
        if new_current == ssid:
            print(f"Successfully connected to {ssid}")
            return True
        else:
            print(f"Failed to connect to {ssid}")
            return False
    except Exception as e:
        print(f"WiFi switch error: {e}")
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
    print("Database ready")

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
        print(f"Last sync for {floor}: {result}")
        return datetime.strptime(result, '%Y-%m-%d %H:%M:%S')
    else:
        print(f"No existing data for {floor}, fetching all")
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
            INSERT OR IGNORE INTO attendance_logs (user_id, timestamp, device_floor, punch_type, is_manually_edited)
            VALUES (?, ?, ?, ?, 0)
        ''', (user_id, timestamp, floor, punch_type))
        conn.commit()
        return cursor.rowcount > 0
    except Exception as e:
        print(f"Error saving log: {e}")
        return False
    finally:
        conn.close()

def fetch_device_data(floor):
    config = WIFI_CONFIG[floor]
    print(f"\n{'='*50}")
    print(f"Fetching data from {floor}")
    print(f"WiFi: {config['ssid']}")
    print(f"Device IP: {config['device_ip']}")
    print(f"{'='*50}")
    
    try:
        if not switch_wifi(config['ssid'], config['password']):
            print(f"Failed to connect to WiFi: {config['ssid']}")
            return 0
        
        print(f"Connecting to device at {config['device_ip']}...")
        zk = ZK(config['device_ip'], port=config['device_port'], timeout=5, password=config['device_password'], force_udp=False, ommit_ping=True)
        conn = zk.connect()
        
        if not conn:
            print(f"Cannot connect to device at {config['device_ip']}")
            return 0
        
        print("Connected to device successfully!")
        
        try:
            conn.disable_device()
        except:
            pass
        
        print("Fetching users...")
        users = conn.get_users()
        user_count = 0
        for user in users:
            if user.name:
                save_user_to_db(str(user.user_id), user.name, floor)
                user_count += 1
        print(f"Saved {user_count} users")
        
        last_sync = get_last_sync_time(floor)
        print(f"Fetching logs newer than {last_sync}...")
        all_logs = conn.get_attendance()
        
        new_count = 0
        for log in all_logs:
            if log.timestamp > last_sync:
                if save_attendance_to_db(str(log.user_id), log.timestamp, floor, log.punch):
                    new_count += 1
        
        try:
            conn.enable_device()
        except:
            pass
        
        conn.disconnect()
        print(f"SUCCESS: {new_count} NEW records saved from {floor}")
        return new_count
        
    except Exception as e:
        print(f"Error fetching from {floor}: {e}")
        return 0

def main():
    print("\n" + "="*60)
    print("ATTENDANCE FETCHER - Incremental Mode")
    print("Manually edited records will NOT be overwritten")
    print("="*60)
    
    current_wifi = get_current_wifi()
    print(f"Starting WiFi: {current_wifi}")
    
    init_database()
    
    print("\n>>> STEP 1: Fetching 2nd Floor Data")
    floor2_new = fetch_device_data('2nd_floor')
    
    print("\n>>> STEP 2: Fetching 3rd Floor Data")
    floor3_new = fetch_device_data('3rd_floor')
    
    final_wifi = get_current_wifi()
    
    conn = sqlite3.connect('attendance.db')
    cursor = conn.cursor()
    cursor.execute("SELECT COUNT(*) FROM users")
    user_count = cursor.fetchone()[0]
    cursor.execute("SELECT COUNT(*) FROM attendance_logs")
    log_count = cursor.fetchone()[0]
    cursor.execute("SELECT COUNT(*) FROM attendance_logs WHERE is_manually_edited = 1")
    edited_count = cursor.fetchone()[0]
    conn.close()
    
    print("\n" + "="*60)
    print("FETCH COMPLETE!")
    print(f"2nd Floor new records: {floor2_new}")
    print(f"3rd Floor new records: {floor3_new}")
    print(f"Total NEW records added: {floor2_new + floor3_new}")
    print(f"Total Users in DB: {user_count}")
    print(f"Total Attendance Logs in DB: {log_count}")
    print(f"Manually Edited Records (protected): {edited_count}")
    print(f"Final WiFi connection: {final_wifi}")
    print("="*60)

if __name__ == '__main__':
    main()

    ******************************************************************************************************************************************************************************************************************
    [edit_app.py](https://github.com/user-attachments/files/28502579/edit_app.py)
from flask import Flask, render_template_string, request, jsonify, session, redirect, url_for
from datetime import datetime
import sqlite3
from functools import wraps

app = Flask(__name__)
app.secret_key = 'final_edit_app_key'
app.config['SESSION_PERMANENT'] = True

def login_required(f):
    @wraps(f)
    def decorated(*args, **kwargs):
        if not session.get('logged_in'):
            return redirect(url_for('login'))
        return f(*args, **kwargs)
    return decorated

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

LOGIN_PAGE = """
<!DOCTYPE html>
<html>
<head>
    <title>Login</title>
    <style>
        body { font-family: Arial; background: #667eea; display: flex; justify-content: center; align-items: center; height: 100vh; margin: 0; }
        .box { background: white; padding: 40px; border-radius: 10px; width: 350px; }
        input { width: 100%; padding: 10px; margin: 10px 0; }
        button { width: 100%; padding: 10px; background: #667eea; color: white; border: none; cursor: pointer; }
        .error { color: red; margin-bottom: 10px; }
        h2 { text-align: center; }
    </style>
</head>
<body>
<div class="box">
    <h2>AL USMAN ENTERPRISE</h2>
    <form method="POST">
        {% if error %}<div class="error">{{ error }}</div>{% endif %}
        <input type="password" name="access_code" placeholder="Access Code" required>
        <button type="submit">Login</button>
    </form>
</div>
</body>
</html>
"""

EDIT_PAGE = """
<!DOCTYPE html>
<html>
<head>
    <title>Edit Attendance</title>
    <style>
        body { font-family: Arial; background: #f0f2f5; padding: 20px; }
        .container { max-width: 1400px; margin: auto; background: white; border-radius: 10px; padding: 20px; }
        .header { background: linear-gradient(135deg, #667eea, #764ba2); color: white; padding: 15px; text-align: center; border-radius: 10px 10px 0 0; margin: -20px -20px 20px -20px; }
        .sub-header { display: flex; justify-content: space-between; background: #667eea; color: white; padding: 10px 20px; margin: 0 -20px 20px -20px; }
        .filter-section { background: #f8f9fa; padding: 15px; border-radius: 8px; margin-bottom: 20px; display: flex; gap: 10px; flex-wrap: wrap; align-items: flex-end; }
        .filter-group { display: flex; flex-direction: column; }
        .filter-group input, .filter-group select { padding: 8px; border: 1px solid #ddd; border-radius: 5px; width: 150px; }
        .btn { padding: 8px 20px; border: none; border-radius: 5px; cursor: pointer; }
        .btn-primary { background: #667eea; color: white; }
        .btn-secondary { background: #6c757d; color: white; }
        .btn-edit { background: #ff9800; color: white; padding: 5px 15px; border: none; border-radius: 3px; cursor: pointer; }
        .btn-save { background: #28a745; color: white; }
        .btn-cancel { background: #6c757d; color: white; }
        table { width: 100%; border-collapse: collapse; }
        th, td { border: 1px solid #ddd; padding: 8px; text-align: left; }
        th { background: #f8f9fa; }
        .sunday-row { background: #fff3e0; }
        .saturday-row { background: #e3f2fd; }
        .modal { display: none; position: fixed; z-index: 1000; left: 0; top: 0; width: 100%; height: 100%; background: rgba(0,0,0,0.5); }
        .modal-content { background: white; margin: 10% auto; width: 400px; border-radius: 10px; }
        .modal-header { background: #667eea; color: white; padding: 15px; border-radius: 10px 10px 0 0; display: flex; justify-content: space-between; }
        .modal-body { padding: 20px; }
        .modal-footer { padding: 15px; border-top: 1px solid #ddd; display: flex; justify-content: flex-end; gap: 10px; }
        .close { cursor: pointer; font-size: 20px; }
        .form-group { margin-bottom: 15px; }
        .form-group label { display: block; margin-bottom: 5px; font-weight: bold; }
        .form-group input { width: 100%; padding: 8px; border: 1px solid #ddd; border-radius: 5px; }
        .logout-btn { background: rgba(255,255,255,0.2); padding: 8px 20px; border: none; border-radius: 5px; color: white; cursor: pointer; }
    </style>
</head>
<body>
<div class="container">
    <div class="header"><h1>AL USMAN ENTERPRISE</h1></div>
    <div class="sub-header"><span>Edit Attendance System</span><button class="logout-btn" onclick="logout()">Logout</button></div>
    
    <div class="filter-section">
        <div class="filter-group"><label>From Date</label><input type="date" id="from_date" min="2026-05-01"></div>
        <div class="filter-group"><label>To Date</label><input type="date" id="to_date" min="2026-05-01"></div>
        <div class="filter-group"><label>Device</label><select id="device"><option value="all">All</option><option value="2nd_floor">2nd Floor</option><option value="3rd_floor">3rd Floor</option></select></div>
        <div class="filter-group"><label>Employee</label><input type="text" id="employee_name" placeholder="Search"></div>
        <div><button class="btn btn-primary" onclick="loadData()">Apply</button></div>
        <div><button class="btn btn-secondary" onclick="resetFilters()">Reset</button></div>
    </div>
    
    <div style="overflow-x:auto">
        <table id="attendance_table">
            <thead><tr><th>Date</th><th>Day</th><th>Device</th><th>Employee</th><th>Check In</th><th>Check Out</th><th>Action</th></tr></thead>
            <tbody id="table_body"><tr><td colspan="7">Loading...</td></tr></tbody>
        </table>
    </div>
</div>

<div id="editModal" class="modal">
    <div class="modal-content">
        <div class="modal-header"><span>Edit Record</span><span class="close" onclick="closeModal()">&times;</span></div>
        <div class="modal-body">
            <input type="hidden" id="edit_id">
            <div class="form-group"><label>Employee</label><input type="text" id="edit_employee" readonly></div>
            <div class="form-group"><label>Date</label><input type="text" id="edit_date" readonly></div>
            <div class="form-group"><label>Device</label><input type="text" id="edit_device" readonly></div>
            <div class="form-group"><label>Check In</label><input type="text" id="edit_checkin" placeholder="Enter any value"></div>
            <div class="form-group"><label>Check Out</label><input type="text" id="edit_checkout" placeholder="Enter any value"></div>
        </div>
        <div class="modal-footer"><button class="btn btn-cancel" onclick="closeModal()">Cancel</button><button class="btn btn-save" onclick="saveEdit()">Save</button></div>
    </div>
</div>

<script>
    let records = [];
    const today = new Date();
    const seven = new Date(); seven.setDate(today.getDate()-7);
    const minDate = new Date('2026-05-01');
    document.getElementById('from_date').value = (seven < minDate ? minDate : seven).toISOString().split('T')[0];
    document.getElementById('to_date').value = today.toISOString().split('T')[0];
    
    loadData();
    
    async function loadData() {
        const params = new URLSearchParams({
            from_date: document.getElementById('from_date').value,
            to_date: document.getElementById('to_date').value,
            device: document.getElementById('device').value,
            employee_name: document.getElementById('employee_name').value
        });
        try {
            const resp = await fetch('/get_attendance?' + params);
            const data = await resp.json();
            records = data.records;
            const tbody = document.getElementById('table_body');
            tbody.innerHTML = '';
            for (let r of records) {
                let rowClass = '';
                if (r.day === 'Sunday') rowClass = 'sunday-row';
                else if (r.day === 'Saturday') rowClass = 'saturday-row';
                let editBtn = r.is_sunday ? '<button disabled style="opacity:0.5">Edit</button>' : '<button class="btn-edit" onclick="openEdit(' + r.id + ')">Edit</button>';
                tbody.innerHTML += '<tr class="' + rowClass + '">' +
                    '<td>' + r.date + '</td>' +
                    '<td>' + r.day + '</td>' +
                    '<td>' + r.device + '</td>' +
                    '<td>' + r.employee_name + '</td>' +
                    '<td>' + r.check_in + '</td>' +
                    '<td>' + r.check_out + '</td>' +
                    '<td>' + editBtn + '</td>' +
                '</tr>';
            }
        } catch(e) { console.error(e); }
    }
    
    function openEdit(id) {
        const r = records.find(x => x.id == id);
        if (r) {
            document.getElementById('edit_id').value = r.id;
            document.getElementById('edit_employee').value = r.employee_name;
            document.getElementById('edit_date').value = r.date;
            document.getElementById('edit_device').value = r.device;
            document.getElementById('edit_checkin').value = r.check_in !== '---' ? r.check_in : '';
            document.getElementById('edit_checkout').value = r.check_out !== '---' ? r.check_out : '';
            document.getElementById('editModal').style.display = 'block';
        }
    }
    
    async function saveEdit() {
        const id = document.getElementById('edit_id').value;
        let checkin = document.getElementById('edit_checkin').value;
        let checkout = document.getElementById('edit_checkout').value;
        
        if (checkin === '') checkin = null;
        if (checkout === '') checkout = null;
        
        try {
            const resp = await fetch('/update_record', {
                method: 'POST',
                headers: {'Content-Type': 'application/json'},
                body: JSON.stringify({id: id, check_in: checkin, check_out: checkout})
            });
            const result = await resp.json();
            if (result.success) {
                alert('Saved successfully!');
                closeModal();
                loadData();
            } else {
                alert('Error: ' + result.message);
            }
        } catch(e) {
            alert('Error: ' + e.message);
        }
    }
    
    function closeModal() { document.getElementById('editModal').style.display = 'none'; }
    function resetFilters() { location.reload(); }
    function logout() { if(confirm('Logout?')) window.location.href = '/logout'; }
</script>
</body>
</html>
"""

@app.route('/login', methods=['GET', 'POST'])
def login():
    if request.method == 'POST':
        code = request.form.get('access_code')
        if code == 'alusman@123':
            session['logged_in'] = True
            return redirect(url_for('index'))
        return render_template_string(LOGIN_PAGE, error='Invalid code')
    return render_template_string(LOGIN_PAGE)

@app.route('/logout')
def logout():
    session.clear()
    return redirect(url_for('login'))

@app.route('/')
@login_required
def index():
    return render_template_string(EDIT_PAGE)

@app.route('/get_attendance')
@login_required
def get_attendance():
    from_date = request.args.get('from_date')
    to_date = request.args.get('to_date')
    device = request.args.get('device')
    employee = request.args.get('employee_name', '')
    
    conn = get_db()
    query = "SELECT a.*, u.name as user_name FROM attendance_logs a LEFT JOIN users u ON a.user_id = u.user_id AND a.device_floor = u.device_floor WHERE DATE(a.timestamp) >= '2026-05-01'"
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
    conn.close()
    
    records = {}
    for log in logs:
        date = log['timestamp'].split()[0]
        key = f"{log['user_id']}_{date}"
        if key not in records:
            records[key] = {
                'id': log['id'],
                'date': date,
                'name': log['user_name'] or f"User {log['user_id']}",
                'device': '2nd Floor' if log['device_floor'] == '2nd_floor' else '3rd Floor',
                'check_in': None, 'check_out': None
            }
        if log['punch_type'] == 0:
            records[key]['check_in'] = datetime.strptime(log['timestamp'], '%Y-%m-%d %H:%M:%S')
        else:
            records[key]['check_out'] = datetime.strptime(log['timestamp'], '%Y-%m-%d %H:%M:%S')
    
    if employee:
        records = {k:v for k,v in records.items() if employee.lower() in v['name'].lower()}
    
    result = []
    for r in records.values():
        day_name = datetime.strptime(r['date'], '%Y-%m-%d').strftime('%A')
        check_in_str = r['check_in'].strftime('%H:%M') if r['check_in'] else '---'
        check_out_str = r['check_out'].strftime('%H:%M') if r['check_out'] else '---'
        result.append({
            'id': r['id'],
            'date': format_date(r['date']),
            'day': day_name,
            'device': r['device'],
            'employee_name': r['name'],
            'check_in': check_in_str,
            'check_out': check_out_str,
            'is_sunday': day_name == 'Sunday'
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
        
        cursor.execute('SELECT user_id, timestamp, device_floor, punch_type FROM attendance_logs WHERE id = ?', (record_id,))
        current = cursor.fetchone()
        if not current:
            conn.close()
            return jsonify({'success': False, 'message': 'Record not found'})
        
        user_id, old_timestamp, device_floor, punch_type = current
        date_part = old_timestamp.split()[0]
        day_name = datetime.strptime(date_part, '%Y-%m-%d').strftime('%A')
        
        if day_name == 'Sunday':
            conn.close()
            return jsonify({'success': False, 'message': 'Sunday records cannot be edited'})
        
        cursor.execute('SELECT id, punch_type FROM attendance_logs WHERE user_id = ? AND DATE(timestamp) = ? AND device_floor = ?', (user_id, date_part, device_floor))
        existing = cursor.fetchall()
        
        check_in_id = None
        check_out_id = None
        for ex in existing:
            if ex[1] == 0:
                check_in_id = ex[0]
            else:
                check_out_id = ex[0]
        
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
                    cursor.execute('UPDATE attendance_logs SET timestamp = ?, is_manually_edited = 1 WHERE id = ?', (new_time, check_in_id))
                else:
                    cursor.execute('INSERT INTO attendance_logs (user_id, timestamp, device_floor, punch_type, is_manually_edited) VALUES (?, ?, ?, ?, ?)', (user_id, new_time, device_floor, 0, 1))
            except:
                pass
        
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
                    cursor.execute('UPDATE attendance_logs SET timestamp = ?, is_manually_edited = 1 WHERE id = ?', (new_time, check_out_id))
                else:
                    cursor.execute('INSERT INTO attendance_logs (user_id, timestamp, device_floor, punch_type, is_manually_edited) VALUES (?, ?, ?, ?, ?)', (user_id, new_time, device_floor, 1, 1))
            except:
                pass
        
        conn.commit()
        conn.close()
        return jsonify({'success': True, 'message': 'Updated successfully'})
        
    except Exception as e:
        return jsonify({'success': False, 'message': str(e)})

if __name__ == '__main__':
    app.run(host='127.0.0.1', port=5015, debug=False)
