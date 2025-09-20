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
# Install required packages
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
```

**Copy and paste the complete HTML code from the secure website above** (the one with the blue gradient design and file upload interface)

**Save and exit** (Ctrl+X, then Y, then Enter)

## **Step 4: Create Startup Scripts**

### **Create basic start script:**
```bash
nano start.sh
```

**Copy and paste:**
```bash
#!/data/data/com.termux/files/usr/bin/bash

clear
echo "🚀 Starting File Upload Server..."
echo "=================================="
echo ""

# Check if Node.js is installed
if ! command -v node &> /dev/null; then
    echo "❌ Node.js is not installed!"
    echo "Please install with: pkg install nodejs -y"
    exit 1
fi

# Check if node_modules exists
if [ ! -d "node_modules" ]; then
    echo "📦 Installing dependencies..."
    npm install
fi

# Create uploads directory if it doesn't exist
mkdir -p uploads

# Get local IP address
LOCAL_IP=$(ip route get 1.1.1.1 2>/dev/null | grep -oP 'src \K\S+' || echo "localhost")

echo "✅ All checks passed!"
echo ""
echo "📍 Server will be accessible at:"
echo "   Local:   http://localhost:3000"
echo "   Network: http://$LOCAL_IP:3000"
echo ""
echo "🔒 Security Info:"
echo "   - Admin password required for text editing"
echo "   - Unlimited file uploads"
echo "   - Rate limiting enabled"
echo ""
echo "🛑 Press Ctrl+C to stop the server"
echo "=================================="
echo ""

# Start the server
exec node server.js
```

**Make it executable:**
```bash
chmod +x start.sh
```

## 🌐 **Step 5: Install Cloudflare Tunnel (Optional)**

### **Install cloudflared(in proot btw) :**
```bash
# Download for Android ARM64
wget https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-arm64

# Make executable and move to bin
chmod +x cloudflared-linux-arm64
./cloudflared-linux-arm64 tunnel --url localhost:3000
```


**Copy and paste:**
```bash
#!/data/data/com.termux/files/usr/bin/bash

clear
echo "🚀 Starting File Upload Server with Public Access..."
echo "===================================================="
echo ""

# Start the web server in background
echo "🌐 Starting web server..."
node server.js &
SERVER_PID=$!


**Make it executable:**
```bash
chmod +x start-with-tunnel.sh
```

## **Step 6: First Run**

### **Start your server:**
```bash
# Basic local server
./start.sh

# OR with public tunnel
./start-with-tunnel.sh
```

### **Test it works:**
1. Open browser and go to `http://localhost:3000`
2. You should see the upload interface
3. Try uploading a small file
4. Check it appears in the file list

##  **Step 7: Security Setup**

### **Change the admin password:**
```bash
# Set a secure password
export ADMIN_PASSWORD="your-super-secure-password-here"

# Start server with custom password
./start.sh
```

### **Or edit server.js directly:**
```bash
nano server.js
# Find line: const ADMIN_PASSWORD = process.env.ADMIN_PASSWORD || 'admin123';
# Change 'admin123' to your secure password
```

## **Step 8: File Management**

### **Create file management script:**
```bash
nano manage-files.sh
```

**Copy the file management script from the earlier artifact**

**Make executable:**
```bash
chmod +x manage-files.sh
```

**Use it:**
```bash
./manage-files.sh list    # List all files
./manage-files.sh size    # Check storage usage
./manage-files.sh clean   # Clean old files
```  

### **Access Your Server:**
- **Local:** `http://localhost:3000`
- **Network:** `http://YOUR-PHONE-IP:3000`
- **Public:** Via Cloudflare tunnel URL

### **Admin Features:**
- **Edit website text** by clicking gray areas
- **Password-protected** changes
- **File deletion** capabilities

## 🛠️ **Troubleshooting**

### **Server won't start:**
```bash
# Check if port is in use
netstat -tulpn | grep :3000

# Try different port
PORT=8080 node server.js
```

### **Can't access from other devices:**
```bash
# Make sure you're using 0.0.0.0 not localhost
# Check your IP address
ip route get 1.1.1.1
```

### **Upload fails:**
```bash
# Check uploads directory exists
ls -la uploads/

# Check disk space
df -h
```

## 🎯 **Quick Commands Summary**

```bash
# Basic setup
pkg update && pkg upgrade -y
pkg install nodejs npm -y
mkdir ~/file-upload-server
cd ~/file-upload-server

# Start server
./start.sh

# With public access
./start-with-tunnel.sh

# Manage files
./manage-files.sh

# Check what's running
ps aux | grep node
```
