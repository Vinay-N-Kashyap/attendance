const express = require('express');
const path = require('path');
const mysql = require('mysql2/promise');
const cors = require('cors');

const app = express();
const PORT = process.env.PORT || 3000;

app.use(cors());
app.use(express.json());
app.use(express.static(path.join(__dirname, 'public')));

const pool = mysql.createPool({
  host: 'localhost',
  user: 'root',
  password: 'vinayrocker2002@',
  database: 'attendance_db',
  waitForConnections: true,
  connectionLimit: 10,
  queueLimit: 0
});

// Get all courses with class counts
app.get('/courses', async (req, res) => {
  try {
    const [rows] = await pool.query(`
      SELECT c.course_id, c.course_code, c.course_name,
             COUNT(cl.class_code) AS total_classes
      FROM courses c
      LEFT JOIN classes cl ON c.course_id = cl.course_id
      GROUP BY c.course_id, c.course_code, c.course_name
      ORDER BY c.course_code
    `);
    res.json(rows);
  } catch (err) {
    console.error('DB error:', err);
    res.status(500).json({ error: 'Database error' });
  }
});

// Add new course
app.post('/add-course', async (req, res) => {
  const { course_code, course_name } = req.body;
  if (!course_code || !course_name) return res.status(400).json({ error: 'Missing fields' });
  try {
    await pool.query('INSERT INTO courses (course_code, course_name) VALUES (?, ?)', 
      [course_code, course_name]);
    res.json({ success: true });
  } catch (err) {
    console.error('DB error:', err);
    res.status(500).json({ error: 'Database error' });
  }
});

// Delete course with cascade
app.delete('/delete-course/:course_id', async (req, res) => {
  const { course_id } = req.params;
  const conn = await pool.getConnection();
  try {
    await conn.beginTransaction();
    
    // Get all class codes for this course
    const [classes] = await conn.query('SELECT class_code FROM classes WHERE course_id = ?', [course_id]);
    
    // Delete attendance for all classes in this course
    for (const cls of classes) {
      await conn.query('DELETE FROM attendance WHERE class_code = ?', [cls.class_code]);
      await conn.query('DELETE FROM enrollments WHERE class_code = ?', [cls.class_code]);
    }
    
    // Delete classes
    await conn.query('DELETE FROM classes WHERE course_id = ?', [course_id]);
    
    // Finally delete course
    await conn.query('DELETE FROM courses WHERE course_id = ?', [course_id]);
    
    await conn.commit();
    res.json({ success: true });
  } catch (err) {
    await conn.rollback();
    console.error('DB error:', err);
    res.status(500).json({ error: 'Database error' });
  } finally {
    conn.release();
  }
});

// Get all classes with course info
app.get('/classes', async (req, res) => {
  try {
    const [rows] = await pool.query(`
      SELECT cl.class_code, cl.class_name, cl.year, cl.section,
             c.course_id, c.course_code, c.course_name,
             COUNT(e.student_id) AS total_students
      FROM classes cl
      JOIN courses c ON cl.course_id = c.course_id
      LEFT JOIN enrollments e ON cl.class_code = e.class_code
      GROUP BY cl.class_code, cl.class_name, cl.year, cl.section,
               c.course_id, c.course_code, c.course_name
      ORDER BY c.course_code, cl.year, cl.section
    `);
    res.json(rows);
  } catch (err) {
    console.error('DB error:', err);
    res.status(500).json({ error: 'Database error' });
  }
});

// Add new class
app.post('/add-class', async (req, res) => {
  const { course_id, year, section } = req.body;
  if (!course_id || !year || !section) return res.status(400).json({ error: 'Missing fields' });
  
  try {
    // Get course info
    const [courses] = await pool.query('SELECT course_code, course_name FROM courses WHERE course_id = ?', [course_id]);
    if (courses.length === 0) return res.status(404).json({ error: 'Course not found' });
    
    const course = courses[0];
    const class_code = `${course.course_code}-${year}-${section}`;
    const yearSuffix = year == 1 ? '1st' : year == 2 ? '2nd' : '3rd';
    const class_name = `${course.course_code} ${yearSuffix} Year Section ${section}`;
    
    await pool.query(
      'INSERT INTO classes (class_code, class_name, course_id, year, section) VALUES (?, ?, ?, ?, ?)',
      [class_code, class_name, course_id, year, section]
    );
    res.json({ success: true });
  } catch (err) {
    console.error('DB error:', err);
    if (err.code === 'ER_DUP_ENTRY') {
      res.status(400).json({ error: 'This class already exists' });
    } else {
      res.status(500).json({ error: 'Database error' });
    }
  }
});

// Delete class with cascade
app.delete('/delete-class/:class_code', async (req, res) => {
  const { class_code } = req.params;
  const conn = await pool.getConnection();
  try {
    await conn.beginTransaction();
    
    await conn.query('DELETE FROM attendance WHERE class_code = ?', [class_code]);
    await conn.query('DELETE FROM enrollments WHERE class_code = ?', [class_code]);
    await conn.query('DELETE FROM classes WHERE class_code = ?', [class_code]);
    
    await conn.commit();
    res.json({ success: true });
  } catch (err) {
    await conn.rollback();
    console.error('DB error:', err);
    res.status(500).json({ error: 'Database error' });
  } finally {
    conn.release();
  }
});

// Add new student and enrollment
app.post('/add-student', async (req, res) => {
  const { student_id, student_name, class_code } = req.body;
  if (!student_id || !student_name || !class_code) return res.status(400).json({ error: 'Missing fields' });

  const conn = await pool.getConnection();
  try {
    await conn.beginTransaction();
    
    await conn.query(`
      INSERT INTO students (student_id, student_name) 
      VALUES (?, ?)
      ON DUPLICATE KEY UPDATE student_name = VALUES(student_name)
    `, [student_id, student_name]);
    
    await conn.query(`
      INSERT IGNORE INTO enrollments (student_id, class_code) 
      VALUES (?, ?)
    `, [student_id, class_code]);
    
    await conn.commit();
    res.json({ success: true });
  } catch (err) {
    await conn.rollback();
    console.error('DB error:', err);
    res.status(500).json({ error: 'Database error' });
  } finally {
    conn.release();
  }
});

// Delete student from class with cascade
app.delete('/delete-student/:student_id/:class_code', async (req, res) => {
  const { student_id, class_code } = req.params;
  const conn = await pool.getConnection();
  try {
    await conn.beginTransaction();
    
    await conn.query('DELETE FROM attendance WHERE student_id = ? AND class_code = ?', [student_id, class_code]);
    await conn.query('DELETE FROM enrollments WHERE student_id = ? AND class_code = ?', [student_id, class_code]);
    
    await conn.commit();
    res.json({ success: true });
  } catch (err) {
    await conn.rollback();
    console.error('DB error:', err);
    res.status(500).json({ error: 'Database error' });
  } finally {
    conn.release();
  }
});

// Get students for a class
app.get('/students/:class_code', async (req, res) => {
  const { class_code } = req.params;
  try {
    const [rows] = await pool.query(`
      SELECT s.student_id, s.student_name
      FROM students s
      JOIN enrollments e ON s.student_id = e.student_id
      WHERE e.class_code = ?
      ORDER BY s.student_id
    `, [class_code]);
    res.json(rows);
  } catch (err) {
    console.error('DB error:', err);
    res.status(500).json({ error: 'Database error' });
  }
});

// Get attendance status for a student in a class on a specific date
app.get('/attendance/:student_id/:class_code/:date', async (req, res) => {
  const { student_id, class_code, date } = req.params;
  try {
    const [rows] = await pool.query(`
      SELECT status FROM attendance
      WHERE student_id = ? AND class_code = ? AND attendance_date = ?
    `, [student_id, class_code, date]);
    res.json(rows[0] || null);
  } catch (err) {
    console.error('DB error:', err);
    res.status(500).json({ error: 'Database error' });
  }
});

// Mark attendance for a specific date
app.post('/mark-attendance', async (req, res) => {
  const { student_id, class_code, status, date } = req.body;
  
  if (!student_id || !class_code || !status) {
    return res.status(400).json({ error: 'Missing fields' });
  }

  const attendanceDate = date || new Date().toISOString().split('T')[0];
  
  try {
    await pool.query(`
      INSERT INTO attendance (student_id, class_code, attendance_date, status)
      VALUES (?, ?, ?, ?)
      ON DUPLICATE KEY UPDATE status = VALUES(status)
    `, [student_id, class_code, attendanceDate, status.toLowerCase()]);
    
    res.json({ success: true });
  } catch (err) {
    console.error('DATABASE ERROR:', err);
    res.status(500).json({ error: 'Database error', details: err.message });
  }
});

// Get student attendance summary per class
app.get('/student-attendance/:student_id', async (req, res) => {
  const { student_id } = req.params;
  try {
    const [rows] = await pool.query(`
      SELECT c.class_code, c.class_name, co.course_code,
             COUNT(a.attendance_date) AS total_classes,
             SUM(a.status = 'present') AS present_count,
             ROUND(100 * SUM(a.status = 'present') / NULLIF(COUNT(a.attendance_date), 0), 2) AS attendance
      FROM enrollments e
      JOIN classes c ON e.class_code = c.class_code
      JOIN courses co ON c.course_id = co.course_id
      LEFT JOIN attendance a ON a.student_id = e.student_id AND a.class_code = c.class_code
      WHERE e.student_id = ?
      GROUP BY c.class_code, c.class_name, co.course_code
      ORDER BY co.course_code, c.year, c.section
    `, [student_id]);
    res.json(rows);
  } catch (err) {
    console.error('DB error:', err);
    res.status(500).json({ error: 'Database error' });
  }
});

// Get detailed attendance history for a student in a specific class
app.get('/student-attendance-history/:student_id/:class_code', async (req, res) => {
  const { student_id, class_code } = req.params;
  try {
    const [rows] = await pool.query(`
      SELECT attendance_date, status
      FROM attendance
      WHERE student_id = ? AND class_code = ?
      ORDER BY attendance_date DESC
      LIMIT 30
    `, [student_id, class_code]);
    res.json(rows);
  } catch (err) {
    console.error('DB error:', err);
    res.status(500).json({ error: 'Database error' });
  }
});

// Dashboard stats
app.get('/stats', async (req, res) => {
  try {
    const [[coursesCount]] = await pool.query('SELECT COUNT(*) total_courses FROM courses');
    const [[classesCount]] = await pool.query('SELECT COUNT(*) total_classes FROM classes');
    const [[studentsCount]] = await pool.query('SELECT COUNT(*) total_students FROM students');
    const [[attendanceCount]] = await pool.query('SELECT COUNT(*) total_attendance FROM attendance');
    res.json({
      total_courses: coursesCount.total_courses,
      total_classes: classesCount.total_classes,
      total_students: studentsCount.total_students,
      total_attendance: attendanceCount.total_attendance
    });
  } catch (err) {
    console.error('DB error:', err);
    res.status(500).json({ error: 'Database error' });
  }
});

// Test database connection
app.get('/test-db', async (req, res) => {
  try {
    const [rows] = await pool.query('SELECT 1 + 1 AS result');
    res.json({ status: 'Database connected successfully', result: rows[0].result });
  } catch (err) {
    console.error('Database connection error:', err);
    res.status(500).json({ error: 'Database connection failed', details: err.message });
  }
});

app.listen(PORT, () => {
  console.log(`========================================`);
  console.log(`Server running at http://localhost:${PORT}`);
  console.log(`Test database: http://localhost:${PORT}/test-db`);
  console.log(`Make sure your database is set up with the enhanced schema!`);
  console.log(`========================================`);
});

