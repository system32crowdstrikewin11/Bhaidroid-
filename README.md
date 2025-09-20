# Bhaidroid-

## **Step 1: Create Project Directory**

```bash
# Create main directory
mkdir ~/file-upload-server
cd ~/file-upload-server

# Initialize npm project
npm init -y
```

##  **Step 2: Install Dependencies**

```bash
# Install required packagesj
npm install express multer cors helmet express-rate-limit

# Verify installation
npm list
```

##  **Step 3: Create Server Files**

### **Create server.js:**
```bash
nano server.js
```

**Copy and paste this entire server code:**

```javascript
const express = require('express');
const multer = require('multer');
const path = require('path');
const fs = require('fs');
const cors = require('cors');
const helmet = require('helmet');
const rateLimit = require('express-rate-limit');

const app = express();
const PORT = process.env.PORT || 3000;
const ADMIN_PASSWORD = process.env.ADMIN_PASSWORD || 'admin123'; // Change this!

// Security middleware
app.use(helmet({
    contentSecurityPolicy: false,
    crossOriginEmbedderPolicy: false
}));

app.use(cors());
app.use(express.json());
app.use(express.static('public'));

// Rate limiting - increased for media sharing
const uploadLimiter = rateLimit({
    windowMs: 15 * 60 * 1000, // 15 minutes
    max: 50, // limit each IP to 50 uploads per windowMs
    message: 'Too many uploads from this IP, please try again later.'
});

const adminLimiter = rateLimit({
    windowMs: 15 * 60 * 1000, // 15 minutes
    max: 5, // limit each IP to 5 admin requests per windowMs
    message: 'Too many admin requests from this IP, please try again later.'
});

// Create uploads directory if it doesn't exist
const uploadsDir = path.join(__dirname, 'uploads');
if (!fs.existsSync(uploadsDir)) {
    fs.mkdirSync(uploadsDir, { recursive: true });
}

// Create config file for storing editable text
const configFile = path.join(__dirname, 'config.json');
if (!fs.existsSync(configFile)) {
    const defaultConfig = {
        welcome_message: "Welcome to my unlimited media sharing service! Upload any files - massive videos, photo albums, documents, etc. No size limits, no restrictions!"
    };
    fs.writeFileSync(configFile, JSON.stringify(defaultConfig, null, 2));
}

// Configure multer for file uploads
const storage = multer.diskStorage({
    destination: (req, file, cb) => {
        cb(null, uploadsDir);
    },
    filename: (req, file, cb) => {
        // Sanitize filename
        const sanitized = file.originalname.replace(/[^a-zA-Z0-9.-]/g, '_');
        const timestamp = Date.now();
        cb(null, `${timestamp}_${sanitized}`);
    }
});

const upload = multer({
    storage: storage,
    limits: {
        fileSize: Infinity, // No file size limit
        files: 20 // Maximum 20 files per upload
    },
    fileFilter: (req, file, cb) => {
        // Block only the most dangerous file types, allow all media files including AV1
        const dangerousExtensions = ['.exe', '.bat', '.cmd', '.scr', '.pif', '.com', '.vbs', '.jar'];
        const ext = path.extname(file.originalname).toLowerCase();
        
        if (dangerousExtensions.includes(ext)) {
            return cb(new Error('File type not allowed'));
        }
        
        cb(null, true);
    }
});

// Middleware to check admin password
const checkAdminAuth = (req, res, next) => {
    const authHeader = req.headers.authorization;
    const password = authHeader && authHeader.split(' ')[1];
    
    if (password !== ADMIN_PASSWORD) {
        return res.status(401).json({ error: 'Unauthorized' });
    }
    
    next();
};

// Routes

// Serve the main HTML file
app.get('/', (req, res) => {
    try {
        const config = JSON.parse(fs.readFileSync(configFile, 'utf8'));
        
        let html = fs.readFileSync(path.join(__dirname, 'index.html'), 'utf8');
        
        html = html.replace(
            'Welcome to my unlimited media sharing service! Upload any files - massive videos, photo albums, documents, etc. No size limits, no restrictions!',
            config.welcome_message || 'Welcome to my unlimited media sharing service! Upload any files - massive videos, photo albums, documents, etc. No size limits, no restrictions!'
        );
        
        res.send(html);
    } catch (error) {
        res.status(500).send('Error loading page');
    }
});

// Upload endpoint
app.post('/upload', uploadLimiter, upload.array('file'), (req, res) => {
    if (!req.files || req.files.length === 0) {
        return res.status(400).json({ error: 'No files uploaded' });
    }
    
    const uploadedFiles = req.files.map(file => ({
        originalName: file.originalname,
        filename: file.filename,
        size: file.size,
        uploadDate: new Date().toISOString()
    }));
    
    console.log(`Files uploaded: ${uploadedFiles.map(f => f.originalName).join(', ')}`);
    res.json({ 
        message: 'Files uploaded successfully', 
        files: uploadedFiles 
    });
});

// Get list of files
app.get('/files', (req, res) => {
    try {
        const files = fs.readdirSync(uploadsDir).map(filename => {
            const filepath = path.join(uploadsDir, filename);
            const stats = fs.statSync(filepath);
            
            const originalName = filename.replace(/^\d+_/, '');
            
            return {
                name: originalName,
                filename: filename,
                size: stats.size,
                date: stats.mtime.toISOString()
            };
        });
        
        files.sort((a, b) => new Date(b.date) - new Date(a.date));
        
        res.json(files);
    } catch (error) {
        res.status(500).json({ error: 'Error reading files' });
    }
});

// Download endpoint - optimized for direct file access and sharing
app.get('/download/:filename', (req, res) => {
    try {
        const requestedName = decodeURIComponent(req.params.filename);
        
        const files = fs.readdirSync(uploadsDir);
        const actualFile = files.find(f => f.replace(/^\d+_/, '') === requestedName);
        
        if (!actualFile) {
            return res.status(404).json({ error: 'File not found' });
        }
        
        const filepath = path.join(uploadsDir, actualFile);
        const stats = fs.statSync(filepath);
        const ext = path.extname(requestedName).toLowerCase();
        
        // Enhanced MIME types for all media formats
        const mimeTypes = {
            // Videos - all formats supported
            '.mp4': 'video/mp4',
            '.webm': 'video/webm', 
            '.mov': 'video/quicktime',
            '.avi': 'video/x-msvideo',
            '.mkv': 'video/x-matroska',
            '.wmv': 'video/x-ms-wmv',
            '.flv': 'video/x-flv',
            '.m4v': 'video/x-m4v',
            '.3gp': 'video/3gpp',
            '.av01': 'video/av01',
            // Images - all formats
            '.jpg': 'image/jpeg',
            '.jpeg': 'image/jpeg',
            '.png': 'image/png',
            '.gif': 'image/gif',
            '.webp': 'image/webp',
            '.avif': 'image/avif',
            '.heic': 'image/heic',
            '.heif': 'image/heif',
            '.bmp': 'image/bmp',
            '.tiff': 'image/tiff',
            '.svg': 'image/svg+xml',
            // Audio
            '.mp3': 'audio/mpeg',
            '.wav': 'audio/wav',
            '.ogg': 'audio/ogg',
            '.aac': 'audio/aac',
            '.flac': 'audio/flac',
            '.m4a': 'audio/mp4',
            '.wma': 'audio/x-ms-wma',
            // Documents
            '.pdf': 'application/pdf',
            '.txt': 'text/plain',
            '.doc': 'application/msword',
            '.docx': 'application/vnd.openxmlformats-officedocument.wordprocessingml.document',
            '.zip': 'application/zip',
            '.rar': 'application/x-rar-compressed'
        };
        
        const mimeType = mimeTypes[ext] || 'application/octet-stream';
        
        // Set basic headers for all files
        res.setHeader('Content-Type', mimeType);
        res.setHeader('Content-Length', stats.size);
        res.setHeader('Cache-Control', 'public, max-age=86400');
        
        // For media files - serve inline for direct viewing
        if (ext.match(/\.(mp4|webm|mov|avi|mkv|wmv|flv|m4v|3gp|av01|jpg|jpeg|png|gif|webp|avif|heic|heif|bmp|tiff|svg|mp3|wav|ogg|aac|flac|m4a|wma)$/i)) {
            res.setHeader('Content-Disposition', `inline; filename="${requestedName}"`);
            
            // Add range support for video/audio streaming
            if (ext.match(/\.(mp4|webm|mov|avi|mkv|wmv|flv|m4v|3gp|av01|mp3|wav|ogg|aac|flac|m4a|wma)$/i)) {
                res.setHeader('Accept-Ranges', 'bytes');
                
                // Handle range requests for streaming
                const range = req.headers.range;
                if (range) {
                    const parts = range.replace(/bytes=/, "").split("-");
                    const start = parseInt(parts[0], 10);
                    const end = parts[1] ? parseInt(parts[1], 10) : stats.size - 1;
                    
                    if (start >= stats.size) {
                        res.status(416).setHeader('Content-Range', `bytes */${stats.size}`);
                        return res.end();
                    }
                    
                    const chunksize = (end - start) + 1;
                    
                    res.status(206);
                    res.setHeader('Content-Range', `bytes ${start}-${end}/${stats.size}`);
                    res.setHeader('Content-Length', chunksize);
                    
                    const fileStream = fs.createReadStream(filepath, { start, end });
                    fileStream.pipe(res);
                    
                    console.log(`Streaming ${requestedName} (${start}-${end}/${stats.size})`);
                    return;
                }
            }
        } else {
            // For documents and other files - force download
            res.setHeader('Content-Disposition', `attachment; filename="${requestedName}"`);
        }
        
        // Stream the entire file
        const fileStream = fs.createReadStream(filepath);
        fileStream.pipe(res);
        
        const fileType = ext.match(/\.(jpg|jpeg|png|gif|webp|avif|heic|heif|bmp|tiff|svg)$/i) ? 'image' :
                         ext.match(/\.(mp4|webm|mov|avi|mkv|wmv|flv|m4v|3gp|av01)$/i) ? 'video' :
                         ext.match(/\.(mp3|wav|ogg|aac|flac|m4a|wma)$/i) ? 'audio' : 'file';
        
        console.log(`${fileType} served: ${requestedName} (${stats.size} bytes)`);
    } catch (error) {
        console.error('Download error:', error);
        res.status(500).json({ error: 'Error downloading file' });
    }
});

// Admin endpoint to update text
app.post('/admin/update-text', adminLimiter, checkAdminAuth, (req, res) => {
    try {
        const updates = req.body;
        const config = JSON.parse(fs.readFileSync(configFile, 'utf8'));
        
        Object.keys(updates).forEach(key => {
            config[key] = updates[key];
        });
        
        fs.writeFileSync(configFile, JSON.stringify(config, null, 2));
        
        console.log('Text updated by admin');
        res.json({ message: 'Text updated successfully' });
    } catch (error) {
        res.status(500).json({ error: 'Error updating text' });
    }
});

// Admin endpoint to delete files
app.delete('/admin/files/:filename', adminLimiter, checkAdminAuth, (req, res) => {
    try {
        const requestedName = decodeURIComponent(req.params.filename);
        const files = fs.readdirSync(uploadsDir);
        const actualFile = files.find(f => f.replace(/^\d+_/, '') === requestedName);
        
        if (!actualFile) {
            return res.status(404).json({ error: 'File not found' });
        }
        
        const filepath = path.join(uploadsDir, actualFile);
        fs.unlinkSync(filepath);
        
        console.log(`File deleted by admin: ${requestedName}`);
        res.json({ message: 'File deleted successfully' });
    } catch (error) {
        res.status(500).json({ error: 'Error deleting file' });
    }
});

// Error handling middleware
app.use((error, req, res, next) => {
    if (error instanceof multer.MulterError) {
        if (error.code === 'LIMIT_FILE_SIZE') {
            return res.status(400).json({ error: 'File too large (max 50MB)' });
        }
        if (error.code === 'LIMIT_FILE_COUNT') {
            return res.status(400).json({ error: 'Too many files (max 5)' });
        }
    }
    
    if (error.message === 'File type not allowed') {
        return res.status(400).json({ error: 'File type not allowed' });
    }
    
    console.error(error);
    res.status(500).json({ error: 'Internal server error' });
});

// Start server
app.listen(PORT, '0.0.0.0', () => {
    console.log(`Server running on http://0.0.0.0:${PORT}`);
    console.log(`Uploads directory: ${uploadsDir}`);
    console.log(`Admin password: ${ADMIN_PASSWORD}`);
});

// Graceful shutdown
process.on('SIGINT', () => {
    console.log('\nShutting down server...');
    process.exit(0);
});
```

**Save and exit** (Ctrl+X, then Y, then Enter)

### **Create index.html:**
```bash
nano index.html
``
<body>
    <div class="container">
        <div class="header">
            <h1>📁 Secure File Share</h1>
            <p>Upload and share your files securely</p>
            <div class="editable-text" contenteditable="true" data-key="welcome_message">
                Welcome to my unlimited media sharing service! Upload any files - massive videos, photo albums, documents, etc. No size limits, no restrictions!
            </div>
        </div>

        <div class="upload-area" id="uploadArea">
            <div class="upload-icon">📁</div>
            <div class="upload-text">Drag & Drop files here</div>
            <div class="upload-subtext">or click to browse</div>
            <input type="file" class="file-input" id="fileInput" multiple>
        </div>

        <div class="progress-bar" id="progressBar">
            <div class="progress-fill" id="progressFill"></div>
        </div>

        <button class="btn" onclick="document.getElementById('fileInput').click()">
            📤 Choose Files
        </button>

        <div class="status-message" id="statusMessage"></div>

        <div class="file-list" id="fileList">
            <h3>📋 Uploaded Files</h3>
        </div>

        <div class="admin-section">
            <h3>🔧 Admin Panel</h3>
            <input type="password" class="admin-input" id="adminPassword" placeholder="Admin Password">
            <button class="btn" onclick="updateText()">💾 Update Text</button>
            <button class="btn" onclick="refreshFiles()">🔄 Refresh Files</button>
        </div>
    </div>

    <script>
        // Configuration
        const MAX_FILE_SIZE = Infinity; // No file size limit
    
