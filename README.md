

# Georgi's Emacs customisations

I will upload here selected Emacs code and customisations.

For now, [this page](http://www.maths.manchester.ac.uk/~gb/emacs/index.html ) contains a couple of packages for typesetting mathematics and
text in Bulgarian and Russian.  For example, typing a single key 
adjust the current input method according to the surrounding context (I usually
assign this key to Capslock).

A blog on R, Emacs and related topics can be found at [My musings](https://geobosh.bitbucket.io/)


## Viewing and updating Rd files in Emacs/ESS

Documentation for packages in the statistical system R is produced from Rd
files. For package authors who directly write Rd files, the package Rdpack
provides the function Rdpack::reprompt() to update the contents of an Rd file.
This can happen, for example, when the arguments of a function are changed or
new methods are added to a function or slots to a class.

Recently Duncan Murdoch contributed an RStudio add-in ('Reprompt') for
convenient use of `Rdpack::reprompt()` in RStudio. In Emacs there are various ways
to accomplish similar effect, depending on the user's workflow. 

Here is the function I use to reprompt and view the Rd file in the current
buffer. I will later put other settings from my ".emacs" file which utilise
further wonders of Emacs.

Choose a key to perform the action and put in your ".emacs" file both, the
assignment for the key and the definition of the function `gnb-preview-help`
given below.

The following definition sets the key `` `C-c p' ``

    (global-set-key [?\C-c ?p] 'gnb-preview-help)

This will be easy to remember if you have used  ESS' key `` `C-c C-p' `` to
view Rd files. In general, it is better to define the key only for Rd files
which under ESS are usually in  "Rd-mode". Bear in mind though that if the point
is in the examples section of the Rd file, then the mode is for R source code,
not Rd. 

Below is the definition of the Elisp function `gnb-preview-help` which reprompts
the Rd file and renders it as text help page. The documentation string should be
sufficiently clear (let me know if it is not). The manipulation of the help
buffer is copied from ESS' `Rd-preview-help`. The reason that I don't simply
call it, is that it currently doesn't render Rd macros, such as bibliographic
references with `\insertRef` from Rdpack (not a big deal if you are not using
such macros).

In addition to reprompting and rendering, `gnb-preview-help` ensures that the
buffer and the Rd file have the same content - it saves the buffer if it has
been modified since the last save. If reprompt is called, it also reloads the
file updated by reprompt. 

    (defun gnb-preview-help (&optional also-reprompt)
      "Reprompt and preview the current Rd buffer contents as rendered help.
    Save the current buffer, if it has been modified since last save.
    If the current buffer is not associated with a file, create a
    temporary one in `temporary-file-directory'.
    
    The file is rendered using the R function Rdpack::viewRd(), which
    processes Rd-macros and, in particular, bibliographic references
    created with \insertRef macro from package Rdpack. The rendered
    help page is shown in a separate buffer as `Rd-preview-help'
    does.
    
    With prefix argument, call Rdpack::reprompt() before rendering,
    then update the current buffer. Any output from
    Rdpack::reprompt() is put in the beginning of the help buffer,
    just before the actual help page starts. This is efficient,
    albeit not very pretty. To get only the help page, just invoke
    the command again without the prefix argument.
    
    This function was initially derived from `Rd-preview-help'.
    "
      (interactive "P")
    
      (require 'ess-help)
      (let ((file buffer-file-name)
            (pbuf (get-buffer-create "R Help Preview"))
            del-p
    	(rcmd ""))
        (if (and file (buffer-modified-p))
    	(save-buffer))
    	
        (unless file
          (setq file (make-temp-file "RD_" nil ".Rd"))
          (write-region (point-min) (point-max) file)
          (setq del-p t))
        
        (if also-reprompt
    	(setq rcmd (format "Rdpack::reprompt(infile = \"%s\", filename = TRUE); " file)))
        (setq rcmd (concat rcmd (format "Rdpack::viewRd(infile = \"%s\")\n" file)))
    
        (with-current-buffer pbuf
          (ess-force-buffer-current "R process to use: ")
          (ess-command rcmd pbuf)    
    
          (ess-setq-vars-local ess-r-customize-alist)
          (setq ess-help-sec-regex ess-help-R-sec-regex
    	    ess-help-sec-keys-alist ess-help-R-sec-keys-alist)
          ;; mostly cut'n'paste from ess--flush-help* (see FIXME(2)):
          (ess-help-underline)
          (ess-help-mode)
          (goto-char (point-min))
          (set-buffer-modified-p 'nil)
          (setq buffer-read-only t)
          (setq truncate-lines nil)
          (when del-p (delete-file file))
          (unless (get-buffer-window pbuf 'visible)
    	(display-buffer pbuf t)))
        (unless (verify-visited-file-modtime)
          (revert-buffer t t))
        ))

