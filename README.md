import { Component, OnInit } from '@angular/core';
import { FormBuilder, FormGroup, Validators } from '@angular/forms';
import { NgbActiveModal } from '@ng-bootstrap/ng-bootstrap';

@Component({
  selector: 'app-contact-form',
  templateUrl: './contact-form.component.html',
  styleUrls: ['./contact-form.component.css']
})
export class ContactFormComponent implements OnInit {
  contactForm: FormGroup;

  constructor(private fb: FormBuilder, public activeModal: NgbActiveModal) {}

  ngOnInit(): void {
    // Initialize the form with default values
    this.contactForm = this.fb.group({
      name: ['', [Validators.required, Validators.minLength(3)]],
      phone: ['', [Validators.required, Validators.pattern(/^\d{10}$/)]]
    });
  }

  // Save the contact
  saveContact(): void {
    if (this.contactForm.valid) {
      console.log(this.contactForm.value); // Send this data to your service
      this.activeModal.close(this.contactForm.value); // Pass data to parent component
    } else {
      this.contactForm.markAllAsTouched(); // Highlight errors
    }
  }

  // Close the modal without saving
  close(): void {
    this.activeModal.dismiss();
  }
}
