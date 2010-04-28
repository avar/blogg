Title: My non-ELPA Emacs swank-clojure setup
Date: 2010-04-07 21:12
Tags: clojure, Emacs, ELPA, swank-clojure

I stubbornly refuse to use ELPA. I keep
[my ~/.emacs](http://github.com/avar/dotemacs) and
[associated libraries](http://github.com/avar/elisp) in Git, and don't
like deviating from that with ELPA by running the clojure code from an
unversioned directory that isn't automatically there when I set up my
Emacs configuration from Git.

I couldn't find any documentation for setting up Emacs + Clojure + 
Swank without ELPA. What follows are the steps I had to do to get my 
setup working so that I can use both SBCL and Clojure with SLIME. 

First checkout the needed clojure code into my ~/g/elisp library: 

    git submodule add git://github.com/technomancy/slime.git slime 
    git submodule add git://github.com/technomancy/clojure-mode.git clojure-mode 
    git submodule add git://github.com/jochu/swank-clojure.git swank-clojure 
    git submodule add git://github.com/richhickey/clojure.git clojure 
    git submodule add git://github.com/richhickey/clojure-contrib.git clojure 

Build clojure: 

    cd clojure && ant 
    cd clojure-contrib && mvn package 

Add this in my ~/.emacs to load the libraries: 

    (add-to-list 'load-path "~/g/elisp/clojure-mode") 
    (add-to-list 'load-path "~/g/elisp/swank-clojure") 
    (add-to-list 'load-path "~/g/elisp/slime") 
    (add-to-list 'load-path "~/g/elisp/slime/contrib") 

Add autoloads + massive hack to get all of this to work: 

    (autoload 'clojure-mode "clojure-mode" nil t) 
    (autoload 'clojure-test-mode "clojure-test-mode" nil t) 
    (defvar package-activated-list nil "Hack: used in 
`slime-changelog-date' but not defined anywhere") 
    (progn 
      (autoload 'swank-clojure-init "swank-clojure") 
      (autoload 'swank-clojure-slime-mode-hook "swank-clojure") 
      (autoload 'swank-clojure-cmd "swank-clojure") 
      (autoload 'swank-clojure-project "swank-clojure")) 

    (setq clojure-src-root (expand-file-name "~/g/elisp")) 

    ;; Java starves programs by default 
    (setq swank-clojure-extra-vm-args (list "-Xmx1024m")) 

This part is very suboptimal, it would be nice if swank-clojure had 
better support for running out-of-Git so I wouldn't have to do 
this. Maybe it does and I haven't found the relevant bits: 

    (defun clojure-slime-config (&optional src-root) 
      "Hacky copy of slime-clojure's `clojure-slime-config' to do what I want." 

      (if src-root (setq clojure-src-root src-root)) 

      (add-to-list 'load-path (concat clojure-src-root "/slime")) 
      (add-to-list 'load-path (concat clojure-src-root "/slime/contrib")) 
      (add-to-list 'load-path (concat clojure-src-root "/swank-clojure")) 

      (require 'slime-autoloads) 

      (slime-setup '(slime-fancy)) 

      (setq swank-clojure-classpath 
            (list 
             (concat clojure-src-root "/clojure/clojure.jar") 
             ;; Hack: Expand the name of the .jar with some Emacs glob function 
             (concat clojure-src-root "/clojure-contrib/target/clojure-contrib-1.2.0-SNAPSHOT.jar") 
             (concat clojure-src-root "/swank-clojure/src") 
             (concat clojure-src-root "/clojure/test/clojure/test_clojure"))) 
      (eval-after-load 'slime 
        '(progn (require 'swank-clojure) 
                (setq slime-lisp-implementations 
                      (cons `(clojure ,(swank-clojure-cmd) :init 
                                      swank-clojure-init) 
                            (remove-if #'(lambda (x) (eq (car x) 'clojure)) 
                                       slime-lisp-implementations)))))) 

    ;;; Setup clojure 
    (clojure-slime-config) 


Finally set it up to play nice so that I can do M-x run-sbcl or M-x 
run-clojure to run Common Lisp or Clojure: 

    ;; http://groups.google.com/group/clojure/browse_thread/thread/e70ac373b47d7088 
    (eval-after-load 'slime 
      '(progn 
         (add-to-list 'slime-lisp-implementations 
                      '(sbcl ("/usr/bin/sbcl"))))) 

    (defun pre-slime () 
      "Stuff to do before SLIME runs" 
      (clojure-slime-config) 
      (slime-setup)) 

    (defun run-clojure () 
      "Starts clojure in Slime" 
      (interactive) 
      (pre-slime) 
      (slime 'clojure)) 

    (defun run-sbcl () 
      "Starts SBCL in Slime" 
      (interactive) 
      (pre-slime) 
      (slime 'sbcl)) 

Of course all of this was such a pain that I pretty much stopped there 
and haven't actually /done/ anything with clojure aside from a Hello 
World :)
