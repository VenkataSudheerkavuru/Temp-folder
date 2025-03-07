<div class="modal-header">
  <h4 class="modal-title">Add Contact</h4>
  <button type="button" class="btn-close" aria-label="Close" (click)="close()"></button>
</div>
<div class="modal-body">
  <form [formGroup]="contactForm">
    <div class="mb-3">
      <label for="name" class="form-label">Name</label>
      <input
        type="text"
        id="name"
        class="form-control"
        formControlName="name"
        [class.is-invalid]="contactForm.get('name')?.invalid && contactForm.get('name')?.touched"
      />
      <div class="invalid-feedback" *ngIf="contactForm.get('name')?.hasError('required')">
        Name is required.
      </div>
      <div class="invalid-feedback" *ngIf="contactForm.get('name')?.hasError('minlength')">
        Name must be at least 3 characters.
      </div>
    </div>
    <div class="mb-3">
      <label for="phone" class="form-label">Phone</label>
      <input
        type="text"
        id="phone"
        class="form-control"
        formControlName="phone"
        [class.is-invalid]="contactForm.get('phone')?.invalid && contactForm.get('phone')?.touched"
      />
      <div class="invalid-feedback" *ngIf="contactForm.get('phone')?.hasError('required')">
        Phone number is required.
      </div>
      <div class="invalid-feedback" *ngIf="contactForm.get('phone')?.hasError('pattern')">
        Phone number must be 10 digits.
      </div>
    </div>
  </form>
</div>
<div class="modal-footer">
  <button type="button" class="btn btn-primary" (click)="saveContact()">Save</button>
  <button type="button" class="btn btn-secondary" (click)="close()">Close</button>
</div>
