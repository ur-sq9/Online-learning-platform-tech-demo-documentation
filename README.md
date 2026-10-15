# Online Learning Platform

## Demo Video

https://github.com/user-attachments/assets/a3293149-b306-4813-85b3-29fd568e7d9f

A multi-step course enrollment and assignment submission system built with vanilla PHP, demonstrating core web development principles including authentication, security, and MVC architecture.

## Overview

**Learning Platform** is an educational web application that allows students to register, browse courses, enroll in programs, and submit assignments. Built entirely with raw PHP, this project showcases backend development practices including user authentication, CSRF protection, file handling, and secure database interactions.

This is a **technical demonstration** of web development fundamentals rather than a production-ready application.

## Key Features

| Feature | Description |
|---------|-------------|
| **Multi-Step Registration** | Secure user onboarding across three stages with session-based data persistence and server-side validation. |
| **User Authentication** | Login system with optional "Remember Me" functionality using secure token-based cookies. |
| **Course Catalog** | Browse available courses with pagination (6 per page), sorting by name/date/popularity, and ascending/descending order options. |
| **Full-Text Search** | Search courses by title and description keywords while preserving sort and pagination settings. |
| **Assignment Submission** | Students can upload assignment files (PDF, DOCX) with server-side file type and size validation. |
| **Activity Logging** | All user actions logged to a text file with timestamps, user IDs, and activity descriptions. |
| **CSRF Protection** | Token-based protection on all forms (registration, login, enrollment, submissions). |
| **Secure Database Queries** | Prepared statements to prevent SQL injection; input sanitization to prevent XSS attacks. |

## Architecture

### MVC Structure

The application follows the **Model-View-Controller** pattern for clear separation of concerns:

- **Models**: Handle data logic, database queries, and business rules (User, Course, Submission classes).
- **Views**: Manage presentation and user interface templates.
- **Controllers**: Process user input and coordinate between models and views.

### Object-Oriented Design

- **Class-based structure** for Users, Courses, Submissions, and other entities.
- **Inheritance and static methods** to demonstrate OOP principles.
- **Encapsulation** of related functionality within appropriate classes.

⚠️ Important Notes <br>
Due to inexperience at the time. Business logic was called Middleware and the show page was called index. 


## Security Features

### CSRF Protection
All forms include CSRF tokens that are validated server-side before processing data.
```php
// loginController.php - CSRF TOKEN GENERATION
public function index()
    {
        // * TOKEN SHOULD ROTATE ON EVERY GET (PAGELOAD) REQUEST
        $token = $_SESSION['csrf_token'] = bin2hex(random_bytes(32));
        $this->view('login', [
            'token' => $token
        ]);
    }

// login.php
<form action="/login" method="post">
            <div><input type="hidden" name="csrf_token" value="<?= $token ?>"></div>
            
            <div>
                <label for="username">Username:</label> <br>
                <input type="text" id="username" name="username_login" value="<?= $username ?? '' ?>">
                <?php if (isset($errors['username_login'])): ?>
                    <span class="error"><?= $errors['username_login'] ?></span>
                <?php endif ?> 
            </div>
            <div>
                <label for="password">Password:</label> <br>
                <input type="password" id="password" name="password_login">
                <?php if (isset($errors['password_login'])): ?>
                    <span class="error"><?= $errors['password_login'] ?></span>
                <?php endif ?> 
            </div>
    
            <div>
                <input type="checkbox" name="remember_me" id="remember_me">
                <label for="remember_me">Remember me</label>
            </div>
            <div><input type="submit" name="login" value="Login"></div>
            <?php if (isset($errors['login'])): ?>
                    <span class="error"><?= $errors['login'] ?></span>
            <?php endif ?> 
        </form>

```

### Input Validation & Sanitization
- Server-side validation at every registration step.
- User input is sanitized to prevent XSS attacks.
- File uploads validated for type and size.

```php
<?php
namespace App\Middleware;

use App\Middleware\SanitizeMiddleware;
use App\Models\User;
use App\Models;
use Core\Router;

class ValidationMiddleware
{
    // public static array $errors = [];
    private string $currentUrl;

    /**
     * @param array $post POST ARRAY
     * @param array $stepsData INSERT STEPS DATA
     */
    public function validation(array $post, ?array $stepsData = [])
    {
        // * SETTING UP  ---------------------------------------------------------------

        $errors = [];
        $loginData = [];
        $LoginErrors = [];
        $url = trim($_SERVER['REQUEST_URI'], '/');

        if (!isset($_SESSION['form-data'])) {
            $_SESSION['form-data'] = [];
        }

        // * URL  ---------------------------------------------------------------
        if ($url === 'register' || $url === 'login') {
            $this->currentUrl = $url;
        }

        // * LOOPING THROUGH POST ---------------------------------------------------------------
        foreach ($post as $key => $value) {
            $passedEmpty[$key] = true;

            // * IF EMPTY  ---------------------------------------------------------------

            if ($this->currentUrl === 'register') {
                if (empty($value) && $key !== 'agree' && $key !== 'username_login' && $key !== 'password_login') {
                    $msg = '⊗ You must fill in ' . str_replace('_', ' ', $key) . '.';
                    $errors[$key] = $msg;
                    $passedEmpty[$key] = false;
                    $_SESSION['form-data'][$key] = '';

                    // continue;  // if empty then this $key should be skipped from further validation (as this needs filling in first), otherwise it just checks for further code
                    // if not empty. Continue validation
                    // HOWEVER, causes issues when input is initialy not empty (filled in) and then deleted. It still maintains the previous answer as it skips code below to change value
                    // SO Passed empty system implemented
                }

                // * PERSONALIZED  ---------------------------------------------------------------

                if ($_SESSION['form-step'] === 1) {
                    if (($key === 'username') && $passedEmpty[$key] === true) {
                        $_SESSION['form-data'][$key] = SanitizeMiddleware::sanitize($value, 'string');
                    }
                    if ($key === 'email' && $passedEmpty[$key] === true) {
                        if (!filter_var($value, FILTER_VALIDATE_EMAIL)) {
                            $errors[$key] = '⊗ Invalid email format';
                            $_SESSION['form-data'][$key] = SanitizeMiddleware::sanitize($value, 'email');
                        } else {
                            $_SESSION['form-data'][$key] = SanitizeMiddleware::sanitize($value, 'email');
                        }
                    }
                } elseif ($_SESSION['form-step'] === 2) {
                    if ($key === 'full_name') {
                        $_SESSION['form-data'][$key] = SanitizeMiddleware::sanitize($value, 'string');
                    }

                    if ($key === 'birthdate' && $passedEmpty[$key] === true) {
                        $_SESSION['form-data'][$key] = date($value);
                    }
                } elseif ($_SESSION['form-step'] === 3) {
                    if ($key === 'password' && $passedEmpty[$key] === true) {
                        if (strlen($value) < 4) {
                            $errors[$key] = '⊗ Password must be atleast 4 characters long.';
                        } else {
                            $_SESSION['form-data'][$key] = $value;  // ? PASSWORDS DO NOT GO THROUGH SANITIZING (they should be hashed before storage)
                        }
                    }
                    if ($key === 'password_confirmation' && $passedEmpty[$key] === true) {
                        if (!isset($_SESSION['form-data']['password']) || $value !== $_SESSION['form-data']['password']) {
                            $errors[$key] = '⊗ Passwords do not match';
                        } else {
                            $_SESSION['form-data'][$key] = $value;  // ? PASSWORDS DO NOT GO THROUGH SANITIZING (they should be hashed before storage)
                        }
                    }
                    // * CHECKBOX  ---------------------------------------------------------------
                    if (!isset($post['agree'])) {
                        // IF NOT CHECKED -> NO VALUE -> MUST CREATE EMPTY FORM DATA FOR 'AGREE' TO RESET FORM DATA
                        $msg = '⊗ You must agree with Terms and Conditions';
                        $errors['agree'] = $msg;
                        $_SESSION['form-data']['agree'] = '';
                    } elseif (isset($post['agree'])) {
                        // IF CHECKED -> ADD VALUE TO 'AGREE'
                        $_SESSION['form-data']['agree'] = $value;
                    }
                }
            } elseif ($this->currentUrl === 'login') {
                if (empty($value)) {
                    $regex = '/_(login)/';
                    $msg = '⊗ You must fill in ' . preg_replace($regex, '', $key) . '.';
                    $LoginErrors[$key] = $msg;
                    $passedEmpty[$key] = false;
                }

                if (($key === 'username_login') && $passedEmpty[$key] === true) {
                    $loginData[$key] = SanitizeMiddleware::sanitize($value, 'string');
                } elseif ($key === 'password_login' && $passedEmpty[$key] === true) {
                    $loginData[$key] = $value;  // ? PASSWORDS DO NOT GO THROUGH SANITIZING (they should be hashed before storage)
                } elseif ($key === 'remember_me') {
                    $loginData[$key] = $value;
                }
            }
        }
        // * ALL ERRORS AFTER FOREACH LOOP ENDED ---------------------------------------------------------------

        $_SESSION['form-errors'] = $errors;
        // die(print_r($_SESSION['form-errors'])); // ? DEBUG
        $loginSession = [$loginData, $LoginErrors];

        // * CSRF VALIDATION ---------------------------------------------------------------
        if (empty($errors) && ($post['csrf_token'] !== $_SESSION['csrf_token']) && $_SERVER['REQUEST_METHOD'] === 'POST') {
            echo 'ERROR IF TOKEN DOESNT MATCH :' . 'POST: ' . $_POST['csrf_token'] . '<br>' . 'SESSION TOKEN: ' . $_SESSION['csrf_token'] . '<br>';  // ? DEBUG
            die();
        }

        // * PAGE NAVIGATION VALIDATION - REGISTER ---------------------------------------------------------------
        if ($this->currentUrl === 'register') {
            if (isset($post['back']) && ($_SESSION['form-step'] > 1)) {
                $_SESSION['form-step'] = $_SESSION['form-step'] - 1;
            }

            if (empty($errors) && isset($post['submit_final']) && ($_SESSION['form-step'] === $_SESSION['form-steps'])) {
                // * ADD TO DB
                $newUser = new User();
                $newUser->registerUser($_SESSION['form-data']);
                // * LOG
                $storageMiddleware = new StorageMiddleware();
                $msg = '[NEW USER: ' . $_SESSION['form-data']['username'] . ']' . ' registered';
                $storageMiddleware->storeLogs($msg);
                unset(
                    $_SESSION['form-data'],
                    $_SESSION['errors'],
                    $_SESSION['form-step'],
                    $_SESSION['form-steps']
                );
                Router::redirect('login');
            }

            if (empty($errors) && isset($post['next']) && ($_SESSION['form-step'] < $_SESSION['form-steps'])) {
                $_SESSION['form-step'] = $_SESSION['form-step'] + 1;
            }
        }
        // * USER DB CHECK - LOGIN ---------------------------------------------------------------
        if ($this->currentUrl === 'login') {
            if (!empty($loginSession)) {
                return $loginSession;
            }
        }
    }
}

```

### Authentication
- Passwords handled securely with "Remember Me" tokens.
- Session management to maintain user state.
- Prepared statements for all database queries to prevent SQL injection.

```php
<?php
namespace App\Controllers;

use App\Middleware\ValidationMiddleware;
use App\Models\User;
use Core\Controller;
use Traits\Controllers\ResetRegisterFormTrait;

class LoginController extends Controller
{
    use ResetRegisterFormTrait;

    public function __construct()
    {
        $this->resetForm();
    }

    public function index()
    {
        // * TOKEN SHOULD ROTATE ON EVERY GET (PAGELOAD) REQUEST
        $token = $_SESSION['csrf_token'] = bin2hex(random_bytes(32));
        $this->view('login', [
            'token' => $token
        ]);
    }

    public function store()
    {
        // * POST ---------------------------------------------------------------
        $errors = [];
        // die(var_dump($_POST)); // ? DEBUG
        if ($_SERVER['REQUEST_METHOD'] === 'POST') {
            $validationMiddleware = new ValidationMiddleware();

            $result = $validationMiddleware->validation($_POST);
            [$loginData, $errors] = $result;
            // ? -> SAME AS $loginData = $result[0]
            // ? -> SAME AS $errors = $result[1]

            if (empty($errors)) {
                $user = new User();
                $data = [
                    'username_login' => $loginData['username_login'],
                    'password_login' => $loginData['password_login']
                ];

                // * TRY TO LOGIN USER
                $loginResult = $user->loginUser($data);

                if ($loginResult === false) {
                    $errors['login'] = '⊗ Username or password do not match/exist.';
                }
            }
        }

        // * CSRF TOKEN ---------------------------------------------------------------
        $token = $_SESSION['csrf_token'] = bin2hex(random_bytes(32));
        // die($loginData['username_login']); // ? DEBUG
        // * RENDER ---------------------------------------------------------------
        $this->view('login', [
            'username' => $loginData['username_login'] ?? '',
            'password' => $loginData['password_login'] ?? '',
            'errors' => $errors,
            'token' => $token
        ]);
    }
}


```

### File Handling
- Uploaded files stored with unique names on the server.
- Files are not publicly accessible.
- Supported formats: PDF, DOCX.

```php
<?php
namespace App\Middleware;

use App\Models\Submits;

class StorageMiddleware
{
    private int $userId;
    private int $courseId;

    public function __construct(?int $userId = 0, ?int $courseId = 0)
    {
        $this->userId = $userId;
        $this->courseId = $courseId;
    }

    private function moveFiles(string $fileTmp, string $directory, ?string $fileName)
    {
        if (!is_dir($directory)) {
            mkdir($directory, 0777, true);
            // echo 'Directory exists. uploading'; // ? DEBUG
        }
        if (move_uploaded_file($fileTmp, $directory . $fileName)) {
            $filePath = $directory . $fileName;
            $relativeFilePath = str_replace(BASE_DIR . '/', '', $filePath);  // REMOVE ABSOLUTE PATH
            // * ADD TO DB
            $newSubmit = new Submits();
            $newSubmit->insertSubmits($this->userId, $this->courseId, $relativeFilePath);
        }
    }

    public function storeUploads(array $file, string $courseTitle)
    {
        if ($file['error'] === UPLOAD_ERR_OK) {  // If Upload successfull with no errors
            $fileName = $file['name'];
            $fileType = $file['type'];
            $fileTmp = $file['tmp_name'];
            $fileSize = $file['size'];

            $allowedTypes = [
                'application/pdf',
                'application/vnd.openxmlformats-officedocument.wordprocessingml.document',
                'application/vnd.openxmlformats-officedocument.presentationml.presentation',
                'image/jpeg',
                'image/png'
            ];

            if (!in_array($fileType, $allowedTypes)) {
                return 'wrong type';
            }

            $maxFileSize = 1 * 1024 * 1024;
            if ($fileSize > $maxFileSize) {
                return 'file too big';
            }

            $sanitizedFileName = preg_replace('/[^a-zA-Z0-9_.\-]/', '', basename($fileName));  // BASENAME STRIP ALL DIRECTORY PATH INFO - SECURITY MEASURE
            $uniqFileName = 'file_' . uniqid() . '_' . $sanitizedFileName;

            $directory = UPLOADS_DIR . $courseTitle . '/' . 'USER_ID_' . $this->userId . '/';
            $this->moveFiles($fileTmp, $directory, $uniqFileName);
        } else {
            return 'no file';
        }
    }
}

```

### Activity Logging
All user interactions (registration, login attempts, course enrollment, submissions) are logged to a secured text file with timestamps and user identifiers.

```php
    public function storeLogs(string $data)
    {
        $logFile = fopen(LOGS_DIR . 'log.txt', 'a');
        fwrite($logFile, '[' . date('Y-m-d H:i:s') . '] ' . $data . "\n");
        fclose($logFile);
    }
```

## Workflow

### Student Journey

1. **Registration**: Complete three-step form collecting basic info, personal details, and credentials.
2. **Login**: Access account with optional "Remember Me" cookie for persistent sessions.
3. **Browse Courses**: View paginated course list with sorting and search capabilities.
4. **Enroll**: Register for a course.
5. **Submit Assignment**: Upload assignment file once enrolled.
6. **Restrictions**: Once an assignment is submitted, students cannot unenroll or resubmit.

## Technical Stack

- **Language**: PHP (vanilla, no frameworks)
- **Database**: Prepared statements for secure queries
- **Sessions & Cookies**: For authentication and state management
- **File System**: Server-side file storage with validation

## Project Status

**Technology Demo**: This project was developed as a student learning exercise to demonstrate core web development concepts. It is not a finished, production-ready product. The codebase prioritizes educational value and concept illustration over scalability or advanced features.


## Learning Outcomes

This project demonstrates proficiency in:

- **PHP fundamentals** and server-side programming
- **Database security** (SQL injection prevention, prepared statements)
- **Authentication workflows** (login, "Remember Me", sessions)
- **CSRF and XSS protection** techniques
- **File handling** and validation
- **MVC architecture** and separation of concerns
- **OOP principles** (classes, inheritance, encapsulation)
- **Form validation** and user feedback

## Personal Note

Built during my studies to solidify understanding of core web development principles. The emphasis is on demonstrating secure, well-structured code rather than visual polish or advanced features. All requirements have been met and tested for functionality and security.
